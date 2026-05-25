# QwenPaw 安全子系统内部实现原理分析

## 一、架构总览

安全子系统位于 `src/qwenpaw/security/`，由 **3 个独立子模块** + **2 个应用级组件**组成：

| 组件 | 目录 | 职责 | 文件数 | 关键类/函数 |
|---|---|---|---|---|
| Secret Store | `security/secret_store.py` | 加密持久化（Fernet + OS keychain） | 1 | `encrypt()`, `decrypt()`, `_get_master_key()` |
| Skill Scanner | `security/skill_scanner/` | 安装前静态安全扫描 | 12 | `SkillScanner`, `PatternAnalyzer`, `ScanPolicy` |
| Tool Guard | `security/tool_guard/` | 运行时工具调用前置拦截 | 11 | `ToolGuardEngine`, `RuleBasedToolGuardian`, `FilePathToolGuardian`, `ShellEvasionGuardian` |
| Approvals | `app/approvals/` | 内存中待审批生命周期 | 2 | `ApprovalService`, `PendingApproval` |
| Auth | `app/auth.py` | HTTP 认证 + JWT + 中间件 | 1 | `AuthMiddleware`, `authenticate()`, `verify_token()` |
| Backup Policy | `app/backup_endpoint_policy.py` | CSRF + IP 白名单备份保护 | 1 | `apply()` |

---

## 二、secret_store — 加密秘密持久化

**文件：** `src/qwenpaw/security/secret_store.py`（413 行）

### 架构

```
encrypt(plaintext)
  └─ Fernet (AES-128-CBC + HMAC-SHA256)
      └─ master_key 从 _get_master_key() 获取
          ├─ OS keychain (keyring 库)
          ├─ └─ 10s daemon 线程超时防止挂死
          ├─ SECRET_DIR/.master_key 文件 (0o600)
          └─ └─ 生成长度不满足则生成新密钥

decrypt(value)
  └─ 检测 ENC: 前缀
      ├─ 有 → Fernet 解密
      └─ 无 → 返回原始值（兼容未加密数据）
```

### 关键设计

- **双重检查锁单例**：`_master_key_lock` + `_cached_master_key` 全局变量
- **密钥解析优先级**：OS keychain > `.master_key` 文件 > 生成新密钥
- **标记格式**：加密结果添加 `ENC:` 前缀，`is_encrypted()` 检查
- **批量方法**：`encrypt_dict_fields()` / `decrypt_dict_fields()` 用于 `api_key`、`jwt_secret` 等敏感字段
- **优雅降级**：`decrypt()` 失败时返回原始密文，不抛异常
- **重载**：`reload_master_key_from_disk()` 在备份恢复后使缓存失效并同步 keychain

### 集成点

| 调用方 | 用途 |
|---|---|
| `providers/provider_manager.py` | 加密/解密所有 Provider 的 `api_key` |
| `app/auth.py` | 加密/解密 `jwt_secret` |
| `envs/store.py` | 加密/解密环境变量值 |

---

## 三、skill_scanner — 安装前技能安全扫描

**目录：** `src/qwenpaw/security/skill_scanner/`

### 3.1 架构

```
__init__.py: 公开 API
  scan_skill_directory(skill_dir)  ← 对外唯一入口
  compute_skill_content_hash()     ← 白名单校验
  is_skill_whitelisted()           ← 白名单检查
  get_blocked_history()            ← 持久化拦截记录

scanner.py: SkillScanner 类
  scan_skill() → _discover_files() → 各 analyzer.analyze() → 去重
  _default_analyzers(): [PatternAnalyzer]
  mtime 缓存 (LRU 64 条)

analyzers/pattern_analyzer.py
  PatternAnalyzer.analyze()
    ├─ 两遍扫描: 逐行 (fast) → 多行回退
    └─ Policy 过滤: disabled rules / doc-path / code-only / severity override

models.py: Finding, ScanResult, Severity, ThreatCategory, SkillFile

scan_policy.py: ScanPolicy (YAML 驱动)
  from_yaml() → 深度合并用户 YAML 到 default_policy.yaml
```

### 3.2 规则文件 (8 个 YAML, 约 660 行)

| 文件 | 规则数 | 检测目标 |
|---|---|---|
| `command_injection.yaml` | 12 | eval, os.system, shell=True, 路径穿越, SQL 注入, SVG/PDF 脚本 |
| `hardcoded_secrets.yaml` | 7 | AWS/Stripe/Google/GitHub 密钥, JWT, 私钥, 密码 |
| `data_exfiltration.yaml` | 7 | HTTP 外发, Socket 连接, 敏感文件读取, base64+网络 |
| `prompt_injection.yaml` | 5 | 忽略指令, jailbreak, 策略绕过, 泄露系统提示, 含中文模式 |
| `obfuscation.yaml` | 4 | base64 decode-exec 链, 十六进制 blob, XOR, 二进制文件 |
| `unauthorized_tool_use.yaml` | 3 | sudo 包安装, 不可信 pip/npm 源, 系统修改 |
| `social_engineering.yaml` | 2 | 模糊描述, 冒充品牌 |
| `supply_chain.yaml` | 1 | 隐藏的 dotfile 代码文件 |

### 3.3 安全措施

- **路径遍历防护**：跳过符号链接，拒绝 skill 目录外的文件
- **文件限制**：`max_files=500`，`max_file_size=10MB`
- **正则安全**：`_safe_compile()` 限制 pattern 长度 1000 防 ReDoS
- **超时保护**：`scan_skill_directory()` 支持 `timeout` 参数，超时返回 `None`
- **三种模式**：`block`（抛 `SkillScanError`）、`warn`（仅日志）、`off`（无操作）

### 3.4 配置解析

优先级：`QWENPAW_SKILL_SCAN_MODE` 环境变量 > `config.json` > 内置默认

默认策略文件：`security/skill_scanner/data/default_policy.yaml`（242 行）

### 3.5 集成点

| 调用方 | 时机 |
|---|---|
| `cli/skills_cmd.py` | CLI 技能安装/激活 |
| `agents/skill_system/store.py` | `scan_skill_dir_or_raise()` 包装 |
| `agents/skill_system/pool_service.py` | 技能池操作 |
| `agents/skill_system/workspace_service.py` | 工作区技能操作 |
| `app/routers/config.py` | REST 查看/更新扫描配置、拦截历史、白名单 |

---

## 四、tool_guard — 运行时工具调用前置拦截

**目录：** `src/qwenpaw/security/tool_guard/`

### 4.1 架构

```
engine.py: ToolGuardEngine (懒加载单例)
  guard(tool_name, params)
    ├─ FilePathToolGuardian.guard()      # always_run=True
    ├─ RuleBasedToolGuardian.guard()     # YAML 正则匹配参数
    └─ ShellEvasionGuardian.guard()      # 引号逃逸检测 (默认关闭)
  is_denied(tool_name)
  is_guarded(tool_name)
  should_auto_deny_result(result)        # CRITICAL/HIGH 直接拒绝

execution_level.py: ToolExecutionLevel 枚举
  STRICT / SMART (默认) / AUTO / OFF

models.py: GuardFinding, ToolGuardResult, GuardSeverity
  TOOL_GUARD_DENIED_MARK = "tool_guard_denied"

approval.py: ApprovalDecision 枚举
  format_findings_summary()             # 格式化发现摘要

utils.py: 配置解析
  resolve_guarded_tools(): [execute_shell_command, read/write/edit/append_file, ...]
  resolve_denied_tools(): 默认空
  resolve_auto_denied_rules(): 规则 ID 列表
```

### 4.2 三种 Guardian

#### RuleBasedToolGuardian (`guardians/rule_guardian.py`, 757 行)

- YAML 规则驱动的正则匹配器，作用于工具参数
- `GuardRule`：id, tools, params, severity, patterns, exclude_patterns
- `load_rules_from_yaml()` / `load_rules_from_directory()`
- 自定义规则 + 禁用规则 ID 从 `config.json` 加载
- `rm` 特殊处理：从 shell 命令提取目标路径，检查工作区边界，生成中英双语告警

**`dangerous_shell_commands.yaml`**（310 行，19 条规则）：
| 规则 | 检测目标 |
|---|---|
| rm | 递归删除、强制删除、根目录删除、跨工作区删除 |
| mv | 移动到危险位置 |
| dd/fdisk | 磁盘操作 |
| fork_bomb | `:(){ :\|:& };:` 等 |
| curl_pipe_bash | `curl ... \| bash/sh` |
| reverse_shell | bash/telnet/nc 反弹 shell |
| crontab_sudoers | 修改调度/权限文件 |
| chmod_777_root | `chmod 777 /` |
| base64_decode_exec | base64 解码后执行 |
| reboot_shutdown | 系统重启/关机 |
| systemctl | 系统服务管理 |
| pkill_killall | 批量结束进程 |
| sudo_su | 提权操作 |
| ifs_injection | `$IFS` 注入 |
| control_chars | 控制字符 |
| unicode_whitespace | Unicode 空白字符混淆 |
| proc_environ | 读取 `/proc/environ` |
| jq_system | `jq` 调用系统命令 |
| zsh_builtins | zsh 危险内建命令 |

#### FilePathToolGuardian (`guardians/file_guardian.py`, 500 行)

- `always_run=True`：即使非 guarded 工具也运行
- 将敏感路径归一化为绝对路径后进行匹配
- Shell 命令路径提取：解析 shell token、重定向操作符、POSIX + Windows 路径
- 跨平台路径归一化：POSIX 主机上也处理 `ntpath` 格式
- Secret 目录保护：`SECRET_DIR`, `.qwenpaw.secret`, `.copaw.secret`
- 配置项：`security.file_guard.sensitive_files`（config.json）

#### ShellEvasionGuardian (`guardians/shell_evasion_guardian.py`, 592 行)

- 字符级引号状态跟踪器（`_QuoteState` 类）
- 7 项检测（**全部默认关闭**）：

| 检测项 | 说明 |
|---|---|
| `command_substitution` | `$()`, 反引号, `<(())` |
| `obfuscated_flags` | 空引号混淆参数（如 `--""flag`）|
| `backslash_escaped_whitespace` | 反斜杠转义的空白 |
| `backslash_escaped_operators` | 反斜杠转义的管道/分号 |
| `newlines` | shell 命令中的换行符 |
| `comment_quote_desync` | 注释与引号失调 |
| `quoted_newline` | 引号内换行 |

### 4.3 运行时数据流

```
Agent._acting(tool_name, params)
  │
  ├── is_denied(tool_name)? → 直接拒绝 (TOOL_GUARD_DENIED_MARK)
  │
  ├── ToolGuardEngine.guard(tool_name, params)
  │   ├── FilePathToolGuardian.guard()     ← sensitive path check
  │   ├── RuleBasedToolGuardian.guard()    ← YAML regex match
  │   └── ShellEvasionGuardian.guard()     ← obfuscation (opt-in)
  │
  ├── result.is_safe? → 放行，执行工具
  │
  ├── should_auto_deny_result(result)? → 返回 TOOL_GUARD_DENIED_MARK
  │
  └── 其他情况:
      ├── ApprovalService.create_pending(...)
      ├── [等待用户审批: /approve /deny 或 REST]
      │   ├── 通过 → 执行工具
      │   └── 拒绝/超时 → 返回 TOOL_GUARD_DENIED_MARK
      └── cancel_stale_pending_for_tool_call()  ← 重放时清理
```

### 4.5 审批等待与恢复流程

#### 4.5.1 核心同步机制：asyncio.Future

```
Agent 阻塞在 asyncio.Future 上，审批方通过 set_result() 解除阻塞

Future 角色:
  Agent ──→ Future ──→ 审批方 (前端/命令/REST)
               ▲
               │ set_result(ApprovalDecision)
               └─────── 解除 Agent 阻塞
```

- `ApprovalService.create_pending()` 为每次待审批创建一个 `asyncio.Future[ApprovalDecision]`
- `_acting_with_approval()` 中 `await future` **阻塞当前 Agent 协程**
- `resolve_request()` 调用 `future.set_result(decision)` → Agent 立即恢复

#### 4.5.2 阻塞等待时序

文件：`agents/tool_guard_mixin.py:488-610`

```
_acting_with_approval(tool_call)
  │
  ├─ [1] ApprovalService.create_pending(request_id, pending_data)
  │        ├─ future = asyncio.get_event_loop().create_future()
  │        └─ _pending[request_id] = PendingApproval(future=future, ...)
  │
  ├─ [2] cancel_stale_pending_for_tool_call(tool_call_id)
  │        ← 清理同一 tool_call_id 的旧待审批（重放场景）
  │
  ├─ [3] _emit_waiting_for_approval_blocking()
  │        ├─ 构造 Msg(message_type="tool_guard_approval")
  │        ├─ metadata: approval_request_id, tool_name, findings, severity
  │        └─ 输出到 channel（SSE event），**不加入记忆**
  │
  ├─ [4] _wait_for_approval_with_heartbeat(future, timeout)
  │        ├─ task = asyncio.create_task(wait_for_future())
  │        ├─ asyncio.wait_for(task, timeout=timeout_seconds)
  │        │    ├─ future.set_result() → 返回 ApprovalDecision
  │        │    ├─ TimeoutError → 返回 ApprovalDecision.TIMEOUT
  │        │    └─ CancelledError → 返回 ApprovalDecision.DENIED
  │        └─ 超时默认 300s (QWENPAW_TOOL_GUARD_APPROVAL_TIMEOUT_SECONDS)
  │
  ├─ [5] 决策分支
  │    ├─ APPROVED → super()._acting(tool_call)  ← 执行真实工具
  │    ├─ DENIED   → _acting_denied() → ToolResultBlock("已拒绝")
  │    └─ TIMEOUT  → _acting_timeout() → ToolResultBlock("已超时")
  │
  └─ [6] 结果加入 agent 记忆，ReAct 循环继续
```

关键点：
- Agent 在步骤 [4] **真正阻塞**，不消耗 CPU
- 步骤 [5] 执行完毕后，ReAct 循环继续下一步 reasoning
- 拒绝/超时产生的 ToolResultBlock 对 Agent 透明，Agent 无法区分是真实的工具返回还是拒绝消息

#### 4.5.3 三条审批恢复路径

| 路径 | 触发方式 | 调用链 | set_result 时机 |
|---|---|---|---|
| **REST API** | 前端 ApprovalCard 点击 | `POST /api/approval/approve` → `approval_router` → `svc.resolve_request()` | 立即 |
| **聊天命令** | 用户输入 `/approval approve <id>` | `query_handler` → `ApprovalCommandHandler` → `svc.resolve_request()` | 立即 |
| **取消/停止** | 用户输入 `/stop` 或 SSE 断开 | `query_handler` 捕获 CancelledError → `svc.cancel_all_pending_by_root_session()` | 批量 DENIED |

##### REST 路径

```python
# app/routers/approval.py:55-90
@router.post("/api/approval/approve")
async def approve_request(body: ApprovalRequest):
    svc = get_approval_service()
    pending = svc._pending.get(body.request_id)
    if pending.root_session_id != body.session_id:
        raise HTTPException(403, "Session mismatch")
    svc.resolve_request(body.request_id, ApprovalDecision.APPROVED)
    # 内部: pending.future.set_result(ApprovalDecision.APPROVED)
    return {"status": "ok"}
```

- 校验 `root_session_id` 匹配，防止跨会话越权
- `resolve_request()` 设置 future 结果 + 将记录移到 completed 列表

##### 命令路径

```
# app/runner/control_commands/approval_handler.py:37-120
/approval approve <request_id>
  └─ ApprovalCommandHandler._handle_approve()
       ├─ 校验 pending 存在
       ├─ 校验 request_id 格式
       ├─ svc.resolve_request(request_id, APPROVED)
       └─ 返回成功消息给用户
```

- 支持快捷命令 `/approve <request_id>` 和 `/deny <request_id>`

##### 取消路径

```python
# app/approvals/service.py:346-370
async def cancel_all_pending_by_root_session(self, root_session_id: str):
    async with self._lock:
        for rid, pending in list(self._pending.items()):
            if pending.root_session_id == root_session_id and not pending.future.done():
                self._completed.append(CompletedApproval(
                    request_id=rid, decision=ApprovalDecision.DENIED, ...
                ))
                pending.future.set_result(ApprovalDecision.DENIED)
```

- `/stop` 时所有待审批被批量拒绝，所有等待的 agent 协程同时恢复

#### 4.5.4 SSE 事件流和前端展示

1. Agent 调用 `_emit_waiting_for_approval_blocking()` 构造 `Msg(message_type="tool_guard_approval")`
2. Channel 层序列化为 SSE event，字段包含：`approval_request_id`、`tool_name`、`findings_summary`、`severity`、`approval_timeout`
3. 前端 `ApprovalCard`（`console/src/components/ApprovalCard/ApprovalCard.tsx`）解析 SSE 数据
4. 渲染为带按钮的审批卡片：工具名 + 风险等级 + 参数摘要 + 倒计时
5. 用户点击 Approve/Deny → `POST /api/approval/approve` 或 `/deny`

#### 4.5.5 跨会话审批（子 Agent 场景）

```
父会话 (root_session_id=A)               子会话 (session_id=B, root_session_id=A)
       │                                       │
       │ 子 Agent 触发审批                       │
       │ ←── PendingApproval(root_session=A) ── │
       │                                       │
       │ 父会话用户审批                          │
       │ REST: session_id=A (root session)      │
       │ 校验: pending.root_session_id == A OK  │
       │                                       │
       │ resolve_request() ── set_result() ──→ │ Agent 解除阻塞
```

- `PendingApproval` 携带 `root_session_id` 和 `agent_id`
- REST API 通过 `X-Root-Session-Id` 头注入 `root_session_id`
- `approval_handler` 额外校验跨会话身份并记录日志

#### 4.5.6 关键设计点

| 设计 | 说明 |
|---|---|
| **纯 asyncio.Future，无消息队列** | 同步阻塞在当前事件循环，无需额外中间件 |
| **审批消息不入 Agent 记忆** | `_emit_waiting_for_approval_blocking()` 仅输出到 channel，防止 Agent 在推理中引用 |
| **重放保护** | `cancel_stale_pending_for_tool_call()` 清理同一 `tool_call_id` 的旧待审批 |
| **GC 策略** | 30min 淘汰过期 pending，上限 200 条，已完成记录上限 500 |
| **线程安全** | `ApprovalService._lock`（asyncio.Lock）保护内部字典 |
| **超时兜底** | 默认 300s 超时自动拒绝，防止 Agent 永久卡死 |

### 4.4 配置

优先层级：环境变量 > `config.json` > 内置默认

| 配置项 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `security.execution_level` | enum | `SMART` | STRICT/SMART/AUTO/OFF |
| `security.tool_guard.guarded_tools` | string[] | 8 个默认工具 | 被守卫的工具列表 |
| `security.tool_guard.denied_tools` | string[] | [] | 被拒绝的工具列表 |
| `security.tool_guard.auto_denied_rules` | string[] | [] | 自动拒绝的规则 ID |

---

## 五、ApprovalService — 审批服务

**文件：** `src/qwenpaw/app/approvals/service.py`（437 行）

### 设计

```python
class ApprovalService:
    _pending: Dict[str, PendingApproval]        # request_id → PendingApproval
    _completed: List[CompletedApproval]         # 已完成的记录（上限 500）
```

- **单例模式**：`_instance` 类变量 + `_lock` 双重检查
- **Future 驱动**：`create_pending()` 创建 `asyncio.Future`，`resolve_request()` 设置结果
- **超时控制**：`wait_for_approval(request_id, timeout)` 带超时的 Future await
- **GC 策略**：30min 淘汰过期 pending，上限 200 条 pending，已完成记录上限 500

### `PendingApproval` 数据类

| 字段 | 说明 |
|---|---|
| `request_id` | 唯一标识（shortuuid） |
| `session_id` / `root_session_id` | 会话路由 |
| `owner_agent_id` / `agent_id` | Agent 标识 |
| `user_id` / `channel` | 用户和频道 |
| `tool_name` | 工具名 |
| `severity` / `findings_count` / `result_summary` | 评分摘要 |
| `timeout_seconds` | 等待超时 |
| `Future` | asyncio.Future 用于 await |

### 集成点

- `agents/tool_guard_mixin.py`：代理拦截工具调用后创建/等待审批
- `app/routers/approval.py`：REST API（审批/拒绝/列表）
- `cancel_all_pending_by_root_session()`：`/stop` 时清理

---

## 六、Authentication — HTTP 认证

**文件：** `src/qwenpaw/app/auth.py`（650 行）

### 设计原则

- **纯 stdlib 实现**：hashlib + hmac + secrets，无 PyJWT/bcrypt 依赖
- **可选启用**：`QWENPAW_AUTH_ENABLED=true` 开启，默认关闭
- **单用户模型**：系统只支持一个用户账户

### 核心函数

| 函数 | 说明 |
|---|---|
| `register_user(username, password)` | 创建账户（16-byte salt + SHA-256 哈希）|
| `authenticate(username, password)` | 验证凭据，签发 token |
| `verify_token(token)` | HMAC 验证 + jti 撤销检查 |
| `create_token(username, expiry)` | 生成：`base64(payload).hmac_signature` |
| `revoke_token(token)` / `revoke_all_tokens()` | 单条/全局撤销 |

### Token 格式

```
base64_url({
  "sub": username,
  "exp": expiry,
  "iat": issued_at,
  "jti": uuid4
}).hex(HMAC-SHA256(base64(payload), jwt_secret))
```

- 默认有效期：7 天（最大 100 年）
- JWT secret 通过 `secret_store` Fernet 加密持久化

### AuthMiddleware

FastAPI `BaseHTTPMiddleware`，对所有请求进行拦截：

- **公开路径**：`/api/auth/login`、`/api/auth/status`、`/api/auth/register`、`/api/version`、`/api/settings/language`
- **公开前缀**：`/assets/`、`/logo.png`、`/qwenpaw-symbol.svg`、`/api/frontend_plugin/`
- **Token 提取**：`Authorization: Bearer <token>` 头 / WebSocket `token` 查询参数
- **本地回环白名单**：`allow_no_auth_hosts`（默认 127.0.0.1, ::1）
- **与其他安全模块集成**：调用 `backup_endpoint_policy.apply()` 做 CSRF 保护
- **自动注册**：通过 `QWENPAW_AUTH_USERNAME`/`QWENPAW_AUTH_PASSWORD` 环境变量实现 Docker 自动部署

---

## 七、BackupEndpointPolicy — 备份端点保护

**文件：** `src/qwenpaw/app/backup_endpoint_policy.py`（215 行）

### 策略

| 场景 | 策略 |
|---|---|
| Auth 启用 + 跨站请求 | 拒绝（CSRF 防护） |
| Auth 启用 + GET `/api/backups/*/export` 来自 localhost | 放行（方便导出） |
| Auth 禁用 | 限制到 `allow_no_auth_hosts` IP 白名单 |
| WebSocket (backup restore) | 要求来自 `sec-fetch-site: same-origin` |

使用 `sec-fetch-site` 头进行 CSRF 检测。配置以 mtime 缓存提高性能。

---

## 八、关键架构决策

1. **抽象基类扩展性**：`BaseAnalyzer`（skill scanner）和 `BaseToolGuardian`（tool guard）采用同一插件模式，新检测器无需修改编排器即可注册

2. **懒加载单例**：`SkillScanner`、`ToolGuardEngine`、`ApprovalService` 均使用双重检查锁单例，线程安全 + 延迟初始化

3. **纯 stdlib 认证**：`app/auth.py` 仅使用 hashlib + hmac + secrets，避免 PyJWT/bcrypt 依赖。独立加密仅隔离在 `secret_store`（Fernet）

4. **配置分层**：全部子系统遵循：`env var > config.json > 默认值`，兼顾灵活性和开箱即用

5. **优雅降级**：`decrypt()` 失败返回原始数据；`SkillScanner` 超时返回 `None`；`FilePathToolGuardian` 路径解析失败不阻断

6. **线程安全**：secret store 双重检查锁；ApprovalService 用 `asyncio.Lock`；扫描结果缓存用常规锁

7. **跨平台路径处理**：`FilePathToolGuardian` 同时处理 POSIX 和 Windows（`ntpath`）路径，即使在 POSIX 主机上也归一化 Windows 路径

8. **中英双语告警**：`rule_guardian.py` 中 `rm` 命令检测对工作区外文件生成中英双语警告

9. **LRU 缓存**：SkillScanner 按 mtime 缓存结果（64 条），减少重复扫描；拦截历史持久化到独立 JSON 文件

10. **Shell 逃逸检测默认关闭**：7 种逃逸检测默认全部 `False`，因高误报率由运维者显式启用

---

## 九、数据流图

```
┌─────────────────────────────────────────────────────────┐
│                     Agent._acting()                      │
│  (agents/tool_guard_mixin.py)                            │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│               ToolGuardEngine.guard()                    │
│  (security/tool_guard/engine.py)                         │
│                                                          │
│  ┌─────────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │ FilePathGuardian │  │ RuleGuardian │  │ ShellEvasion│ │
│  │ (always_run)     │→│ (YAML regex) │→│ (opt-in)    │ │
│  └────────┬────────┘  └──────┬───────┘  └──────┬──────┘ │
│           │                  │                  │        │
│           ▼                  ▼                  ▼        │
│              ToolGuardResult (findings)                   │
└────────────────────────┬────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
    is_safe         auto_deny       needs_approval
        │                │                │
        ▼                ▼                ▼
   执行工具     返回 DENIED      ApprovalService
                                   create_pending()
                                        │
                                        ▼
                              等待审批 /approve /deny
                                        │
                              ┌─────────┴─────────┐
                              ▼                   ▼
                          执行工具           返回 DENIED


┌─────────────────────────────────────────────────────────┐
│                   Secret Store 集成                        │
│                                                          │
│  provider_manager.py ── encrypt/decrypt(api_key)         │
│  app/auth.py ────────── encrypt/decrypt(jwt_secret)      │
│  envs/store.py ──────── encrypt/decrypt(env values)      │
│                                                          │
│  密钥链: OS keychain → .master_key (0o600) → 生成新密钥   │
└─────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────┐
│                   Skill Scanner 集成                      │
│                                                          │
│  CLI install ──→ cli/skills_cmd.py                       │
│  Pool ops ─────→ agents/skill_system/pool_service.py     │
│  Workspace ────→ agents/skill_system/workspace_service   │
│  REST ─────────→ app/routers/config.py (配置/历史/白名单) │
│                                                          │
│  规则引擎: 8 YAML 文件 → PatternAnalyzer → ScanResult    │
│  缓存: LRU 64条, mtime 失效                               │
└─────────────────────────────────────────────────────────┘
```
