# QwenPaw Agent 子系统内部实现原理分析

## 一、架构总览

**目录：** `src/qwenpaw/agents/`

| 子目录/文件 | 行数 | 职责 |
|---|---|---|
| `react_agent.py` | 1462 | `QwenPawAgent` 主 Agent 类 |
| `model_factory.py` | 1075 | 模型 + Formatter 工厂 |
| `tool_guard_mixin.py` | 808 | 工具调用安全拦截 Mixin |
| `command_handler.py` | 748 | `/` 系统命令处理 |
| `skill_system/` | 3100+ | 技能管理系统（5 层模型） |
| `tools/` | ~2500 | 15 个内置工具函数 |
| `memory/` | ~2000 | 记忆系统 + Proactive |
| `prompt.py` | 482 | System Prompt 构建器 |
| `context/` | ~1200 | 上下文管理（token 感知） |
| `mission/` | ~1600 | Mission 自主迭代模式 |
| `acp/` | ~800 | Agent Communication Protocol |
| `utils/` | ~1500 | 工具函数 |
| `hooks/` | ~150 | Bootstrap 钩子 |
| `skills/` | 36 目录 | 18 种技能 × 2 语言 |
| `md_files/` | 5 语言 | 工作区 Markdown 模板 |

### 依赖流向

```
Channel (用户输入)
  │
  ▼
QwenPawAgent.reply()
  ├── 检查系统命令 → CommandHandler
  ├── 设置 workspace_dir 上下文
  ├── apply_skill_config_env_overrides() 注入环境变量
  │
  └── ReActAgent 循环 (super().reply())
      │
      ├── Pre-reasoning hooks (BootstrapHook + ContextManager)
      │
      ├── QwenPawAgent._reasoning()
      │   ├── 主动媒体过滤 (model 能力检查)
      │   ├── ToolGuardMixin._reasoning
      │   ├── ReActAgent._reasoning
      │   │   └── model.__call__ → formatter._format
      │   │       ├── normalize_messages_for_model_request()
      │   │       ├── FileBlockSupportFormatter._format()
      │   │       ├── RetryChatModel + TokenRecordingModelWrapper
      │   │       └── 实际模型 API 调用
      │   ├── Plan 工具门控 (_filter_plan_tools)
      │   └── Auto-continue (纯文本回退)
      │
      ├── _acting 循环 (ToolGuardMixin → QwenPawAgent → ReActAgent)
      │   ├── ToolGuardMixin._acting: 安全检查 (denied/approve/auto)
      │   ├── QwenPawAgent._acting: Plan 门控 + JSON 参数修复
      │   └── Toolkit 执行工具 (skills + builtins + MCP + plugin)
      │
      ├── Post-acting hooks (ContextManager)
      └── Post-reply hooks (ContextManager)
```

---

## 二、QwenPawAgent — 核心 Agent

**文件：** `src/qwenpaw/agents/react_agent.py`（1462 行）

### 2.1 类定义

```python
class QwenPawAgent(ToolGuardMixin, ReActAgent):
```

MRO（方法解析顺序）：`QwenPawAgent → ToolGuardMixin → ReActAgent`。

两者都覆写了 `_acting` 和 `_reasoning`，通过 `super()` 交错调用实现安全拦截与核心逻辑的透明融合。

### 2.2 初始化流程

```python
QwenPawAgent.__init__(AgentProfileConfig, env_context, mcp_clients,
                      memory_manager, context_manager, request_context,
                      namesake_strategy, workspace_dir)
```

| 步骤 | 方法 | 说明 |
|---|---|---|
| 1 | `_resolve_effective_skills(channel)` | 从 workspace manifest 解析渠道匹配的已启用技能 |
| 2 | `_create_toolkit()` | 构建 Toolkit：内置工具 + 插件工具 + MCP 客户端 |
| 3 | `_register_skills()` | 将 workspace skills 注册到 toolkit |
| 4 | `memory_manager` 初始化 | 传入或创建默认记忆管理器 |
| 5 | `context_manager` 初始化 | 传入或创建默认上下文管理器 |
| 6 | `_build_sys_prompt()` | 从 markdown 文件构建 system prompt |
| 7 | `create_model_and_formatter()` | 创建模型 + Formatter |
| 8 | `InMemoryMemory` 创建 | 替换默认内存 |
| 9 | 注册记忆工具 | 从 `memory_manager.list_memory_tools()` |
| 10 | 配置 ContextManager 内存 | `context_manager.set_memory()` |
| 11 | `CommandHandler` 初始化 | 设置系统命令处理器 |
| 12 | `_register_hooks()` | 注册 Bootstrap + ContextManager 生命周期钩子 |
| 13 | MCP 客户端注册 | `register_mcp_clients()` |

### 2.3 关键方法

| 方法 | 说明 |
|---|---|
| `reply(message)` | 主入口。设置 `workspace_dir` 上下文，处理 file/media block，检查系统命令，通过 `apply_skill_config_env_overrides()` 注入技能配置到环境变量，调用 `super().reply()` |
| `_reasoning()` | 主动媒体过滤（text-only model 跳过媒体块、`rejects_media` 缓存标记）。Plan 门控：Plan 变更后强制 `tool_choice="none"`。Auto-continue：纯文本回复时注入提示，最多额外 2 轮推理直到出现工具调用 |
| `_acting()` | Plan 工具门控 `check_plan_tool_gate()`，修复 Plan 工具的 JSON 字符串化参数 `_fix_stringified_json_args()`，管理 `_plan_awaiting_user_confirm` / `_plan_text_only_after_mutation` 状态 |
| `_summarizing()` | 过滤 tool_use block（防止幽灵工具调用），追加 round-end notice |
| `rebuild_sys_prompt()` | 重建 system prompt 并更新内存中的 system message（用于 `load_session_state` 后） |
| `register_mcp_clients()` | 在 Toolkit 上注册 MCP 客户端，带中断会话恢复逻辑 |
| `interrupt()` | 安全取消当前 reply 任务 |

### 2.4 Plan 门控系统

核心变量：
- `_plan_awaiting_user_confirm`：Plan 等待用户确认
- `_plan_text_only_after_mutation`：Plan 变更后的纯文本限制
- `_PLAN_TOOLS_WITH_JSON_ARGS`：需要 JSON 参数修复的 Plan 工具列表
- `_filter_plan_tools()`：过滤并行工具调用中的 Plan 工具
- `_fix_stringified_json_args()`：修复 `json.dumps()` 导致的字符串化参数

Plan 变更后将 `tool_choice` 设为 `"none"`，防止并行工具调用绕过 Plan 门控。

---

## 三、模型与格式化系统

### 3.1 Model Factory (`model_factory.py`, 1075 行)

**核心函数：** `create_model_and_formatter(agent_id) → (ChatModelBase, FormatterBase)`

```python
def create_model_and_formatter(agent_id=None):
    1. 解析 agent_id（参数 > 上下文 > None）
    2. 加载 agent 级模型配置 + 重试/限流设置
    3. 从 agent provider 或全局活跃模型创建 ChatModel
    4. 包裹 TokenRecordingModelWrapper（用量追踪）
    5. 包裹 RetryChatModel（重试/限流/日志）
    6. 创建并包裹 Formatter
```

**`_create_file_block_support_formatter()`：** 从基础 Formatter 创建 `FileBlockSupportFormatter` 子类。

Formatter 包装器功能：
- 清理 tool messages，处理 `thinking` block（推理内容）
- 支持 `file` block 在 `tool_result` 中（对 Anthropic 转为可读文本）
- Video block 占位符替换（用于 OpenAI 兼容 API）
- 将 `tool_result` 的 video 提升为独立 user message
- 重新排序 tool + 提升消息以满足 API 连续性规则
- 去除消息顶层的 `name` 字段
- 修复 MIME 类型（如 `image/jpg` → `image/jpeg`）

**支持的 Formatter 家族：**
| Formatter | 特殊处理 |
|---|---|
| `OpenAIChatFormatter` | `promote_tool_result_images=True` |
| `AnthropicChatFormatter` | 自定义 `_format_anthropic_messages`，完整媒体 block 处理 |
| `GeminiChatFormatter` | `promote_tool_result_images=True` |

### 3.2 模型路由 (`routing_chat_model.py`, 122 行)

| 类 | 功能 |
|---|---|
| `RoutingDecision` | `route` (local/cloud) + `reasons` |
| `RoutingPolicy` | 基于配置模式（`local_first`/`cloud_first`）决策 |
| `RoutingEndpoint` | `provider_id` + `model_name` + `model` + `formatter` |
| `RoutingChatModel` | 在 local/cloud 端点间路由，覆写 `__call__` |

当前路由逻辑为简化模式（仅基于模式配置，无内容感知路由）。

### 3.3 System Prompt 构建 (`prompt.py`, 482 行)

**`PromptConfig`：** `DEFAULT_FILES = ["AGENTS.md", "SOUL.md", "PROFILE.md"]`

**核心函数 `build_system_prompt_from_working_dir()`：**
1. 从工作目录加载 markdown 文件
2. 支持 per-agent 配置（通过 `agent_id`）
3. 心跳段落处理：`<!-- heartbeat:start -->` / `<!-- heartbeat:end -->` 标记。心跳禁用时整个段落移除
4. 记忆段落处理：`<!-- memory:start -->` / `<!-- memory:end -->` 标记。段落总是被移除，替换为记忆管理器的规范记忆提示

**辅助函数：**
- `build_bootstrap_guidance()`：首次设置引导（中英双语）
- `build_multimodal_hint()`：模型多模态能力提示片段
- `get_active_model_supports_multimodal()`：检查活跃模型多模态能力

---

## 四、工具系统

### 4.1 内置工具清单 (`tools/`)

| 工具函数 | 文件 | 说明 |
|---|---|---|
| `read_file` | `file_io.py` | 读文件，支持行范围、智能截断、续读提示 |
| `write_file` | `file_io.py` | 创建/覆写文件，编码检测 |
| `edit_file` | `file_io.py` | 查找替换文本 |
| `append_file` | `file_io.py` | 追加内容 |
| `grep_search` | `file_search.py` | 正则内容搜索 |
| `glob_search` | `file_search.py` | Glob 模式文件搜索 |
| `execute_shell_command` | `shell.py` | 执行 shell 命令（可配置超时、异步） |
| `send_file_to_user` | `send_file.py` | 通过频道发送文件 |
| `browser_use` | `browser_control.py` | CDP 浏览器自动化 |
| `browser_snapshot` | `browser_snapshot.py` | 浏览器截图 |
| `desktop_screenshot` | `desktop_screenshot.py` | 桌面截图 |
| `view_image` / `view_video` | `view_media.py` | 展示媒体给模型 |
| `get_current_time` / `set_user_timezone` | `get_current_time.py` | 时间工具 |
| `get_token_usage` | `get_token_usage.py` | Token 用量报告 |
| `list_agents` / `chat_with_agent` / `submit_to_agent` / `check_agent_task` | `agent_management.py` | Agent 间通信 |
| `delegate_external_agent` | `delegate_external_agent.py` | 委托外部 Agent |
| `materialize_skill` | `make_skill_tools.py` | 创建技能（仅当 `make-skill` 技能启用） |
| `execute_python_code` | 来自 agentscope | 执行 Python 代码 |
| `view_text_file` / `write_text_file` | 来自 agentscope | 遗留文本文件操作 |

**插件工具发现：** `QwenPawAgent._create_toolkit()` 动态扫描 `tools.__all__`，不在硬编码列表中的工具视为插件工具，需要 agent config 中的显式配置条目（安全门控）。

**异步执行：** `execute_shell_command` 和 `delegate_external_agent` 支持 `async_execution=True`，启用后台任务管理（`view_task`、`wait_task`、`cancel_task`）。

### 4.2 工具安全拦截 (`tool_guard_mixin.py`, 808 行)

**类：** `ToolGuardMixin` — MRO mixin，在 `QwenPawAgent` 和 `ReActAgent` 之间。

**`_acting` 拦截流程：**

```
_decide_guard_action(tool_call)
  │
  ├── OFF 模式 → 绕过
  │
  ├── engine.is_denied(tool_name)? → auto_denied
  │
  ├── STRICT 模式 → 即使无发现也创建 INFO finding，全部需审批
  │
  ├── AUTO/SMART 模式:
  │   ├── Guarded tool → 运行 3 Guardian
  │   ├── Non-guarded tool → 仅运行 FilePathToolGuardian (always_run)
  │   ├── SMART: INFO/LOW 自动放行
  │   └── Medium+ → 创建 PendingApproval
  │
  └── 执行决策:
      ├── auto_denied → 发送拒绝响应 (TOOL_GUARD_DENIED_MARK)
      ├── needs_approval → ApprovalService.create_pending()
      │   └── _wait_for_approval_with_heartbeat()
      │       ├── APPROVED → 执行工具
      │       ├── DENIED → 返回拒绝
      │       └── TIMEOUT → 返回超时
      └── auto_allowed → 直接执行
```

**关键方法：**

| 方法 | 说明 |
|---|---|
| `_init_tool_guard()` | 懒加载 GuardEngine 和 ApprovalService |
| `_should_require_approval()` | 检查 session_id 是否存在 |
| `_get_tool_execution_level()` | 从 agent config 读取执行级别 |
| `_wait_for_approval_with_heartbeat()` | 异步等待 + 取消支持 |
| `_emit_waiting_for_approval_blocking()` | 发送审批请求给用户（`message_type: "tool_guard_approval"`） |
| `_acting_auto_denied()` / `_acting_denied()` / `_acting_timeout()` | 各种拒绝响应 |

**本地化：** 通过 `_TOOL_GUARD_I18N` 支持 en/zh/ru/ja 四种语言。

---

## 五、技能系统 (`skill_system/`)

### 5.1 五层模型

```
Packed Builtins (skills/ 目录, 36 个 SKILL.md)
  │  import_builtin_skills()
  ▼
Skill Pool (全局共享池)
  │  SkillPoolService (add/remove/update)
  │  reconcile_pool_manifest()
  ▼
Workspace Manifest (workspace/skill.json)
  │  SkillService (enable/disable/configure)
  │  reconcile_workspace_manifest()
  ▼
Channel Filter (渠道过滤)
  │  resolve_effective_skills(workspace_dir, channel_name)
  ▼
Agent Toolkit (运行时)
  │  toolkit.register_agent_skill()
```

### 5.2 关键组件

| 组件 | 功能 |
|---|---|
| `SkillInfo` (models.py) | 技能元数据 Dataclass |
| `BuiltinSkillIdentity`, `BuiltinSkillVariant` (models.py) | 内置技能名称/语言变体 |
| `resolve_effective_skills(workspace_dir, channel_name)` | 从 workspace manifest 解析渠道匹配的已启用技能 |
| `ensure_skills_initialized(workspace_dir)` | 运行时确保 workspace manifest 存在 |
| `apply_skill_config_env_overrides(workspace_dir, channel_name)` | 上下文管理器：将技能配置注入环境变量（每个 agent turn） |
| `import_builtin_skills()` | 从包源导入内置技能到本地 skill pool |
| `reconcile_pool_manifest()` | 同步 pool 元数据与文件系统 |
| `reconcile_workspace_manifest(workspace_dir)` | 同步 workspace skill.json 与文件系统 |
| `SkillPoolService` | 池级技能管理 |
| `SkillService` | 工作区级技能服务 |
| `read_skill_manifest()` / `read_skill_pool_manifest()` (store.py) | Manifest 文件 I/O |
| `hub.py` (1735 行) | 技能中心客户端：从远端下载/安装技能、冲突解决、YAML 解析 |

### 5.3 环境变量注入机制

`apply_skill_config_env_overrides()` 是一个上下文管理器，在每个 agent turn 时调用：
- 从 workspace 技能配置读取 `env` 字段
- 临时注入到 `os.environ`
- 退出时恢复原始值
- 用于技能需要自定义环境变量（如 API 密钥）的场景

---

## 六、记忆系统 (`memory/`)

### 6.1 架构

```
BaseMemoryManager (ABC)
  │  memory_registry: Registry[BaseMemoryManager]
  │  _summarize_worker() 后台 FIFO 队列
  │
  ├── AgentMdManager
  │     基于 Markdown 文件的长期记忆（读写工作区 MD 文件）
  │
  ├── ReMeLightMemoryManager (默认)
  │     轻量级参考记忆管理器
  │
  └── ADBPGMemoryManager
         AnalyticDB PostgreSQL 后端，大规模记忆检索
```

### 6.2 BaseMemoryManager 接口

| 方法 | 说明 |
|---|---|
| `start()` / `close()` | 生命周期 |
| `get_memory_prompt()` | 获取记忆提示文本 |
| `list_memory_tools()` | 列出注册的记忆工具 |
| `summarize()` | 主动摘要 |
| `retrieve()` | 检索相关记忆 |
| `dream()` | 记忆优化（联想、压缩）|
| `auto_memory_search()` | 自动记忆搜索 |
| `summarize_when_compact()` | 压缩时摘要 |
| `auto_memory()` | 自动记忆 |

**后台摘要工作器：** `_summarize_worker()` 使用 `asyncio.Queue`（FIFO）异步处理摘要任务。

### 6.3 Proactive 子包 (`memory/proactive/`)

| 组件 | 功能 |
|---|---|
| `ProactiveConfig` | 空闲超时、触发参数 |
| `ProactiveTask` | 主动响应任务容器 |
| `enable_proactive_for_session()` | 为会话启用主动模式 |
| `proactive_trigger_loop()` | 后台循环检查空闲时间并触发主动消息 |
| `generate_proactive_response()` | 通过 LLM 生成实际主动响应 |

### 6.4 ADBPGMemoryManager — 云端记忆自动整理

**文件：** `memory/adbpg_memory_manager.py`（508 行）+ `memory/adbpg_client.py`（751 行）

#### 6.4.1 设计哲学

```
Agent 视角:  我是记忆的使用者，不是管理员
                 │
                 ▼
        memory_search → 搜索即可，无需操心写入和整理

系统视角:  每轮对话自动持久化 + ADBPG 服务端自主提取
                 │
                 ▼
  客户端 fire-and-forget  →  服务端 adbpg_llm_memory 扩展
      (不阻塞 Agent)           LLM 提取事实 + Embedder 向量化
```

**与 ReMeLightMemoryManager 的核心区别：**

| 维度 | ReMeLight | ADBPG |
|---|---|---|
| 存储后端 | 本地文件系统 | 云端 AnalyticDB PostgreSQL |
| 写入主体 | Agent 主动写 MEMORY.md / daily notes | 系统自动持久化用户消息 |
| 记忆整理 | DREAM 优化（Agent 运行） | 服务端 LLM 自动提取 |
| 检索方式 | file search + keyword match | 语义向量相似度 |
| Agent 负担 | 高（需主动记录、整理、去重） | 低（只需搜索） |

#### 6.4.2 自动持久化生命周期

```
每轮 Agent reply 结束后 → auto_memory() 触发
  │
  ├─ [1] 过滤: 只取 role=user 的消息
  │         └─ 跳过已持久化的消息（_persisted_msg_ids 去重）
  │
  ├─ [2] _fire_and_forget_add(messages)
  │         └─ daemon thread → client.add_memory()
  │                            ├─ SQL:  SELECT adbpg_llm_memory.add(json)
  │                            └─ REST: POST /v3/memories/add/
  │                                      ↓
  │                             ADBPG 服务端:
  │                               ├─ LLM 从对话文本提取事实/偏好/决策
  │                               ├─ Embedder 生成向量
  │                               └─ 文本 + 向量存入 vector_store
  │
  └─ [3] 更新 _persisted_msg_ids
```

关键特性：

- **interval=1**：每轮都触发（ReMeLight 按配置间隔，ADBPG 无间隔限制）
- **fire-and-forget**：写入不阻塞 Agent，配置中断不丢对话
- **仅持久化 user 消息**（`_filter_user_messages` 过滤 `role=user`），assistant 和 system 内容不存
- **persisted_msg_ids 去重**：`set[str]` 跟踪已发消息 ID，避免重复持久化

#### 6.4.3 记忆提取（服务端，`adbpg_llm_memory` 扩展）

配置（`adbpg_client.py:_configure_connection`）：

```python
config_json = {
    "llm": {
        "provider": "qwen",
        "config": {
            "model": cfg.llm_model,
            "api_key": cfg.llm_api_key,
            "qwen_base_url": cfg.llm_base_url,
        },
    },
    "embedder": {
        "provider": "openai",
        "config": {
            "model": cfg.embedding_model,
            "api_key": cfg.embedding_api_key,
            "embedding_dims": str(cfg.embedding_dims),
            "openai_base_url": cfg.embedding_base_url,
        },
    },
    "vector_store": {
        "provider": "adbpg",
        "config": {
            "port": auto_detected_internal_port,
            "embedding_model_dims": str(cfg.embedding_dims),
        },
    },
}
```

`adbpg_llm_memory` 扩展内部完成（**Python 端不可控**）：
1. 调用 LLM 从原始对话文本中提取事实、用户偏好、重要决策、上下文信息
2. 调用 Embedder 为提取内容生成向量 embedding
3. 存入 ADBPG 的向量存储引擎
4. 支持自定义 fact extraction prompt（`custom_fact_extraction_prompt` 配置项）

#### 6.4.4 自动检索（`auto_memory_search` → `retrieve`）

在 `pre_reply` 阶段触发：

```
retrieve(messages) → auto_memory_search()
  │
  ├─ 从最新 user message 提取 query
  ├─ adbpg_llm_memory.search(query, agent_id, user_id, limit=3)
  │     └─ 语义相似度搜索 → 返回 {content, score} 列表
  │
  └─ 构造 tool_use + tool_result 消息对注入对话
        └─ 搜索结果以 memory_search 工具结果的格式注入
            让 Agent 感知到"系统自动搜索了记忆"
```

注入消息对：

```python
assistant_msg = Msg(role="assistant",
    content=[TextBlock("Searching long-term memory..."),
             ToolUseBlock(name="memory_search", id=_id, input={...})])
tool_result_msg = Msg(role="system",
    content=[ToolResultBlock(id=_id, name="memory_search",
             output=[TextBlock("[Long-term Memory from ADBPG]\n- ...")])])
```

#### 6.4.5 `memory_search` 工具（双重来源）

```python
async def memory_search(query, max_results=5, min_score=0.1):
    parts = []
    # 来源 1: ADBPG 语义搜索
    results = adbpg_llm_memory.search(query, limit=max_results)
    for item in results:
        if item.score >= min_score:
            parts.append(f"[N] (adbpg, score: {score:.2f})\n{content}")

    # 来源 2: 本地文件关键词匹配 (fallback)
    local_hits = keyword_search(MEMORY.md, memory/*.md)
    for filepath, snippet in local_hits:
        parts.append(f"[N] (file: {filepath})\n{snippet}")

    return ToolResponse(parts[:max_results])
```

- ADBPG 结果优先（语义相似度）
- 本地文件作为辅助（关键词命中）
- 结果按评分排序合并

#### 6.4.6 配置与连接管理

| 特性 | 说明 |
|---|---|
| **连接池** | 进程级 `ThreadedConnectionPool`，所有 Agent 共享（`min=2, max=10`）|
| **懒配置** | 每个从池中借出的连接首次使用时执行 `adbpg_llm_memory.config()`，通过 `_configured_conns: set[int]` 按 `id(conn)` 去重 |
| **内部端口自检** | 查询 `gp_segment_configuration` 自动检测 ADBPG master 内部端口 |
| **热重载** | `reset_configured_connections()` 清空配置缓存，下次借出时重新配置（`custom_fact_extraction_prompt` 生效）|
| **Rest 模式** | 可切换为 `api_mode=rest`，走 HTTP POST 到远端 ADBPG memory service，无需本地数据库依赖 |
| **隔离模式** | `memory_isolation` 控制 agent 间记忆隔离（`agent_id` 或 `"shared"`）|
| **超时控制** | `search_timeout`（默认 10s）+ `SET statement_timeout` 防慢查询 |
| **优雅降级** | 配置不完整或连接失败时 client 置为 None，记忆功能静默关闭，不阻断 Agent 运行 |

---

## 七、上下文管理 (`context/`)

### 7.1 架构

```
BaseContextManager (ABC)
  │  context_registry: Registry[BaseContextManager]
  │
  └── LightContextManager
         工具结果裁剪 + 上下文健康检查
         │
         └── AgentContext (extends InMemoryMemory)
               token 感知计数 + JSONL 持久化 + 压缩摘要
```

### 7.2 生命周期钩子

| 钩子 | 时机 | 用途 |
|---|---|---|
| `start()` | Agent 启动 | 初始化 |
| `close()` | Agent 关闭 | 清理 |
| `pre_reply()` | 回复前 | 准备 |
| `pre_reasoning()` | 推理前 | 注入上下文 |
| `post_acting()` | 工具执行后 | 清理工具结果 |
| `post_reply()` | 回复后 | 持久化 |

钩子在 `QwenPawAgent._register_hooks()` 中注册到各个生命周期阶段。

### 7.3 AgentContext

扩展自 AgentScope 的 `InMemoryMemory`，增加了：
- **Token 感知计数**：`add()` 时检测 token 溢出
- **JSONL 持久化**：`persist_dialog()` 将对话持久化到 JSONL 文件
- **压缩摘要**：`update_compressed_summary()` 维护对话压缩摘要
- **历史格式化**：`get_history_str()` 带 token 感知截断的历史格式化

---

## 八、Mission 模式 (`mission/`)

### 8.1 架构

```python
/mission <task_description> [--verify CMD] [--max-iterations N]
  │
  ▼
handle_mission_command()
  ├── 创建循环目录 (mission_<timestamp>/)
  ├── 写入 task.md
  ├── 检测 git 上下文 (branch, remote, diff stat)
  ├── 构建 master prompt
  └── 返回 phase dict
  │
  ▼
MissionRunner
  ├── Phase 1: 主 Agent 生成 PRD.json
  │   └── 任务分解 → PRD（含 goal, steps, files, architecture）
  │
  └── Phase 2: （用户确认后）
      ├── Worker 分发（并行批次）
      │   └── 每个 worker 是独立的 subagent 会话
      └── 验证循环
          ├── verify CMD 执行
          ├── 失败 → 修复 → 重新分发
          └── 成功 → 完成
```

### 8.2 关键组件

| 组件 | 文件 | 功能 |
|---|---|---|
| `handle_mission_command()` | `handler.py` | 解析 `/mission` 命令，构建 phase |
| `MissionRunner` | `mission_runner.py` | Phase 1 + Phase 2 编排 |
| `build_master_prompt()` | `prompts.py` (863 行) | Master prompt：主 Agent 作为"仅控制器，非实现者"，委托工作给 worker 会话 |
| `create_loop_dir()` / `read_prd()` / `detect_git_context()` | `state.py` | 状态文件管理 |

---

## 九、命令处理 (`command_handler.py`, 748 行)

**类：** `CommandHandler(ConversationCommandHandlerMixin)`

**系统命令清单：**

| 命令 | 方法 | 说明 |
|---|---|---|
| `/compact` | `_process_compact()` | 压缩对话为摘要（通过 ContextManager） |
| `/new` | `_process_new()` | 新对话（摘要旧对话，清空 memory） |
| `/clear` | `_process_clear()` | 清空所有 memory 和摘要 |
| `/history` | `_process_history()` | 显示对话历史（token 感知截断） |
| `/compact_str` | `_process_compact_str()` | 显示当前压缩摘要 |
| `/summarize_status` | `_process_summarize_status()` | 显示异步摘要任务状态 |
| `/message <n>` | `_process_message()` | 显示第 n 条消息详情 |
| `/dump_history` | `_process_dump_history()` | 保存历史到 JSONL 文件 |
| `/load_history` | `_process_load_history()` | 从 JSONL 文件加载历史 |
| `/proactive` | `_process_proactive()` | 启用/禁用主动模式 |
| `/plan` | `_process_plan()` | 显示 Plan 状态（`/plan <desc>` 穿透到 Plan 模式） |

**关键设计：** `/plan <description>`（带参数）不是命令——它会穿透到 Plan 模式。只有裸 `/plan` 被视为状态命令。

---

## 十、ACP 协议 (`acp/`)

Agent Communication Protocol，用于分布式 Agent 系统间通信。

| 组件 | 文件 | 功能 |
|---|---|---|
| `ACPErrors` 层次 | `core.py` | `ACPConfigError`、`ACPTransportError`、`ACPProtocolError`、`ACPSessionError` |
| `SuspendedPermission` | `core.py` | 跨 Agent 调用期间权限挂起 |
| ACP Client | `client.py` | 连接远端 ACP Agent |
| `QwenPawACPAgent` | `server.py` | ACP 服务端 Agent 包装器 |
| `run_qwenpaw_agent()` | `server.py` | 在 ACP 模式下运行 Agent |
| `ACPService` | `service.py` | 单例生命周期管理 |
| Permissions | `permissions.py` | 跨 Agent 授权模型 |
| Tool Adapter | `tool_adapter.py` | ACP 工具适配到本地 Toolkit |

---

## 十一、启动钩子 (`hooks/bootstrap.py`)

**类：** `BootstrapHook`

- 注册为 `pre_reasoning` 钩子
- 首次用户交互时检查 `BOOTSTRAP.md`
- 如果存在，将中英双语引导信息预置到第一条用户消息中
- 创建 `.bootstrap_completed` 标记文件防止重复触发

---

## 十二、工具函数 (`utils/`)

### 12.1 Registry[T] 泛型注册模式

**文件：** `utils/registry.py`

```python
class Registry[T]:
    _registry: dict[str, type[T]]

    def register(name=None) → 装饰器
    def get(name) → type[T] | None
    def list_registered() → list[str]
    def has(name) → bool
```

用于 `memory_registry` 和 `context_registry` 的可插拔后端注册。

### 12.2 消息归一化

**`normalize_messages_for_model_request(msgs, supports_multimodal, target_family)`：**
- 深度拷贝消息
- 清理 tool messages
- 剥离 provider 特定字段
- 可选剥离媒体块（text-only model）

### 12.3 工具消息清洗

**`_sanitize_tool_messages()`（tool_message_utils.py）：**
- 修复空的 tool input
- 去重 tool blocks
- 移除无效 block

### 12.4 其他工具

| 函数 | 说明 |
|---|---|
| `process_file_and_media_blocks_in_message()` | 从 base64/URL 下载 file block 并持久化到磁盘 |
| `is_first_user_interaction()` | 检查当前消息是否为首次交互 |
| `prepend_to_message_content()` | 在用户消息前预置引导文本 |
| `download_file_from_url()` / `download_file_from_base64()` | 文件下载 |
| `transcribe_audio()` | Whisper 音频转录 |
| `copy_md_files()` / `copy_template_md_files()` / `copy_workspace_md_files()` | 工作区 markdown 模板填充 |
| `EstimatedTokenCounter` | 无外部 API 的 token 估算 |

---

## 十三、技能资源与模板

### 13.1 内置技能 (`skills/`)

每个技能目录遵循 `<name>-<language>` 模式：
- 18 种技能 × 2 语言（en, zh）= 36 目录
- 复杂技能（`docx`、`pdf`、`pptx`、`xlsx`）包含 `scripts/` 目录

| 技能 | 功能领域 |
|---|---|
| `browser_cdp` / `browser_visible` | 浏览器自动化 |
| `channel_message` | 频道消息 |
| `chat_with_agent` | Agent 间通信 |
| `cron` | 定时任务 |
| `dingtalk_channel` | 钉钉集成 |
| `docx` | Word 文档 |
| `file_reader` | 文件读取 |
| `guidance` | 指导 |
| `himalaya` | 喜马拉雅 |
| `make_plan` | 计划制定 |
| `make-skill` | 技能创建 |
| `multi_agent_collaboration` | 多 Agent 协作 |
| `news` | 新闻 |
| `pdf` | PDF 处理 |
| `pptx` | PowerPoint |
| `QA_source_index` | 问答索引 |
| `xlsx` | Excel |

### 13.2 MD 文件模板 (`md_files/`)

按语言和 Agent 角色组织的 markdown 模板：

| 目录 | 语言 | 文件 |
|---|---|---|
| `md_files/` | en, zh, id, ru | AGENTS.md, BOOTSTRAP.md, HEARTBEAT.md, MEMORY.md, PROFILE.md, SOUL.md |
| `md_files/qa/` | en, zh, ru | AGENTS.md, PROFILE.md, SOUL.md (Q&A Agent 子集) |
| `md_files/local/` | en, zh | SOUL.md (本地 Agent) |

`template_id`（在 `templates.py` 中）决定使用哪个子集：
- `default` → 6 个文件
- `local` → 1 个文件（SOUL.md）
- `qa` → 3 个文件

---

## 十四、完整运行时数据流

```
用户输入 (Channel)
  │
  ▼
QwenPawAgent.reply(message)
  │
  ├── 设置 agent_context
  │   └── get_current_agent_id() → 当前 Agent ID
  │
  ├── 处理 file/media blocks
  │   └── process_file_and_media_blocks_in_message()
  │       ├── base64 → 本地文件
  │       └── URL → 下载到本地
  │
  ├── 检查系统命令
  │   └── command_handler.is_conversation_command()?
  │       ├── 是 → 执行命令处理（/compact, /new, /clear, etc.）
  │       └── 否 → 继续
  │
  ├── 技能环境变量注入
  │   └── with apply_skill_config_env_overrides():
  │       └── super().reply()
  │
  ▼
ReActAgent 循环 (repeated until done or max_rounds):
  │
  ├──== 1. Pre-reasoning hooks ==
  │   ├── BootstrapHook (首次交互引导)
  │   └── ContextManager.pre_reasoning(msgs)
  │
  ├──== 2. _reasoning() ==
  │   ├── ToolGuardMixin._reasoning
  │   │   └── guard-aware reasoning
  │   │
  │   ├── QwenPawAgent._reasoning
  │   │   ├── 媒体过滤:
  │   │   │   ├── 主动: model 能力检查 + rejects_media 缓存
  │   │   │   └── 被动: 模型报错后重试（移除媒体）
  │   │   └── Plan 门控:
  │   │       └── plan 变更后 force tool_choice="none"
  │   │
  │   └── ReActAgent._reasoning
  │       └── model.__call__(msgs, tools, tool_choice)
  │           │
  │           ├── formatter._format(msgs)
  │           │   ├── normalize_messages_for_model_request()
  │           │   │   ├── 深拷贝
  │           │   │   ├── _sanitize_tool_messages()
  │           │   │   └── 可选剥离媒体块
  │           │   │
  │           │   └── FileBlockSupportFormatter._format()
  │           │       ├── tool_result file block 处理
  │           │       ├── video block 占位符替换
  │           │       ├── thinking block 处理
  │           │       ├── tool/promoted msg 重排
  │           │       ├── 去除顶层 name 字段
  │           │       └── MIME 类型修复
  │           │
  │           ├── TokenRecordingModelWrapper
  │           │   └── 记录 token 用量
  │           │
  │           └── RetryChatModel.__call__()
  │               ├── 重试逻辑（临时错误）
  │               └── 限流控制
  │
  ├──== 3. Auto-continue (仅文本时) ==
  │   └── _auto_continue_if_text_only()
  │       ├── 注入提示 "请使用工具..."
  │       └── 最多额外 2 轮推理
  │
  ├──== 4. _acting() ==
  │   │
  │   ├── ToolGuardMixin._acting
  │   │   ├── _decide_guard_action(tool_call)
  │   │   │   ├── OFF → bypass
  │   │   │   ├── denied → auto_denied
  │   │   │   ├── STRICT → 全部需审批
  │   │   │   ├── SMART → INFO/LOW auto, Medium+ 需审批
  │   │   │   └── AUTO → 仅 guarded tools
  │   │   │
  │   │   └── 执行:
  │   │       ├── auto_denied → TOOL_GUARD_DENIED_MARK
  │   │       ├── needs_approval → ApprovalService
  │   │       │   └── _wait_for_approval_with_heartbeat()
  │   │       │       ├── APPROVED → _execute_tool_call()
  │   │       │       ├── DENIED → _acting_denied()
  │   │       │       └── TIMEOUT → _acting_timeout()
  │   │       └── auto_allowed → _execute_tool_call()
  │   │
  │   ├── QwenPawAgent._acting
  │   │   ├── check_plan_tool_gate()
  │   │   ├── _fix_stringified_json_args()
  │   │   └── 管理计划状态标志
  │   │
  │   └── ReActAgent._acting
  │       └── Toolkit.execute(tool_name, params)
  │           ├── Builtin tools（file I/O, shell, browser, etc.）
  │           ├── Plugin tools（通过 tools.__all__ 发现）
  │           ├── Skill tools（通过 SKILL.md 注册）
  │           └── MCP clients（外部工具）
  │
  ├──== 5. Post-acting hooks ==
  │   └── ContextManager.post_acting()
  │       └── 工具结果裁剪 + 上下文健康检查
  │
  ├──== 6. _summarizing() ==
  │   ├── 过滤 tool_use block
  │   └── 追加 round-end notice
  │
  └──== 7. Post-reply hooks ==
      └── ContextManager.post_reply()
          └── JSONL 持久化
```
