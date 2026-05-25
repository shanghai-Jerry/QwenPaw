# QwenPawAgent (react_agent.py) 核心逻辑梳理

**文件：** `src/qwenpaw/agents/react_agent.py`（1462 行）

---

## 一、类层次与 MRO

```python
class QwenPawAgent(ToolGuardMixin, ReActAgent):
```

**MRO：** `QwenPawAgent → ToolGuardMixin → ReActAgent → ReActAgentBase → AgentBase`

| 层级 | 来源 | 关键覆写 |
|---|---|---|
| `QwenPawAgent` | `react_agent.py` | `__init__`, `reply`, `_reasoning`, `_acting`, `_summarizing`, `print`, `rebuild_sys_prompt`, `register_mcp_clients`, `interrupt` |
| `ToolGuardMixin` | `tool_guard_mixin.py` | `_reasoning`（透传）, `_acting`（安全拦截 + 审批） |
| `ReActAgent` | `agentscope` | `reply`（主循环）, `_reasoning`（实际模型调用）, `_acting`（工具执行）, `_summarizing`（兜底） |
| `AgentBase` | `agentscope` | `__call__` → `reply`, `print`, hooks 框架 |

**MRO 设计要点：** 两个父类都覆写 `_acting` 和 `_reasoning`，通过 `super()` 形成交错调用链。`ToolGuardMixin` 的覆写在 MRO 中优先于 `ReActAgent`，因此安全拦截先于实际工具执行。

---

## 二、构造函数：`__init__`（L101-232）

### 2.1 时序

```
__init__(agent_config, env_context, mcp_clients, memory_manager,
        context_manager, request_context, namesake_strategy, workspace_dir)
  │
  ├─ [1] 存储基本属性 (request_context, mcp_clients, etc.)
  │
  ├─ [2] resolve_effective_skills(workspace_dir, channel_name)
  │       └─ 从 workspace manifest 解析渠道匹配的已启用技能
  │
  ├─ [3] _create_toolkit(namesake_strategy, effective_skills)
  │       └─ 构建 Toolkit：内置工具 + 插件工具 + 异步任务管理工具
  │
  ├─ [4] _register_skills(toolkit, effective_skills)
  │       └─ 将 workspace skills 注册到 toolkit
  │
  ├─ [5] _build_sys_prompt()
  │       └─ 从 AGENTS.md / SOUL.md / PROFILE.md 等文件构建 system prompt
  │
  ├─ [6] create_model_and_formatter(agent_id)
  │       └─ 创建模型包装链 + Formatter
  │
  ├─ [7] super().__init__(name, model, sys_prompt, toolkit, memory, formatter, max_iters)
  │       └─ 调用 ReActAgent.__init__，设置基础属性
  │
  ├─ [8] 注册记忆工具 (memory_manager.list_memory_tools)
  │
  ├─ [9] 配置 ContextManager 内存 (self.memory = agent_context)
  │
  ├─ [10] CommandHandler 初始化
  │
  └─ [11] _register_hooks()
          └─ BootstrapHook + ContextManager 生命周期钩子
```

### 2.2 `_create_toolkit` 工具注册逻辑（L234-398）

```
tool_functions 硬编码表 (15+ 内置工具)
  │
  ├─ 从 agent_config.tools.builtin_tools 读取 enabled / async_execution
  │
  ├─ 内置工具默认启用，插件工具需要显式配置（安全门控）
  │
  ├─ async_execution：仅 execute_shell_command / delegate_external_agent 支持
  │
  ├─ 异步工具启用时自动注册 view_task / wait_task / cancel_task
  │
  └─ 条件工具：materialize_skill 仅在 make-skill 技能启用时注册
```

### 2.3 `_register_hooks` 钩子注册（L468-509）

| 钩子类型 | 注册的钩子 | 时机说明 |
|---|---|---|
| `pre_reasoning` | `BootstrapHook` | 首次交互检查 BOOTSTRAP.md |
| `pre_reasoning` | `context_manager.pre_reasoning` | 推理前注入上下文 |
| `pre_reply` | `context_manager.pre_reply` | 回复前准备 |
| `post_acting` | `context_manager.post_acting` | 工具执行后裁剪结果 |
| `post_reply` | `context_manager.post_reply` | 回复后持久化 JSONL |

钩子通过 metaclass `_ReActAgentMeta._wrap_with_hooks` 在方法调用前后自动触发。

---

## 三、入口方法：`reply`（L1385-1446）

```
reply(msg | list[Msg] | None)
  │
  ├─ [1] 设置线程局部上下文
  │       └─ workspace_dir, recent_max_bytes, shell_command_timeout, shell_executable
  │
  ├─ [2] process_file_and_media_blocks_in_message(msg)
  │       └─ 将 base64 / URL 形式的 file block 下载为本地文件
  │
  ├─ [3] command_handler.is_command(query)?
  │       ├─ 是 → 处理系统命令（/compact, /new, /clear, /history, /plan, etc.）
  │       └─ 返回命令执行结果，不走 ReAct 循环
  │
  ├─ [4] apply_skill_config_env_overrides(workspace_dir, channel_name)
  │       └─ 上下文管理器：临时注入技能配置中的环境变量
  │
  └─ [5] return await super().reply(msg, structured_model)
          └─ ReActAgent.reply → 主推理-执行循环
```

**代理入口：** `AgentRunner.query_handler()`（`runner.py:354`）创建 QwenPawAgent 实例后调用 `agent(msgs)`，即 `AgentBase.__call__()`，后者设置 `_reply_task` 后调用 `reply`。

---

## 四、主循环：`ReActAgent.reply`（agentscope 库）

这是由父类实现的核心循环，QwenPawAgent 通过覆写关键方法来影响其行为。

```
for _ in range(max_iters):
    │
    ├─ _compress_memory_if_needed()
    │
    ├─ [pre_reasoning hooks]
    │   └─ BootstrapHook + context_manager.pre_reasoning
    │
    ├─ msg_reasoning = await _reasoning(tool_choice)
    │
    ├─ if msg_reasoning 无 tool_use blocks → break
    │                      (reply_msg = msg_reasoning, 退出循环)
    │
    └─ for each tool_use block in msg_reasoning:
        └─ asyncio.gather(*[self._acting(tc) for tc in tool_use_blocks])
            └─ [post_acting hooks]

if reply_msg is None (耗尽 max_iters 仍未回复):
    reply_msg = await _summarizing()

[post_reply hooks]
return reply_msg
```

---

## 五、`_reasoning`— 模型调用层（L986-1103）

### 5.1 调用链

```
QwenPawAgent._reasoning(tool_choice)
  │
  ├─ [1] Plan 门控：_plan_text_only_after_mutation?
  │       └─ 是 → 强制 tool_choice = "none"
  │
  ├─ [2] 主动媒体过滤
  │       ├─ 检查模型是否支持多模态 (get_active_model_supports_multimodal)
  │       ├─ 检查 capability cache 中 rejects_media 标记
  │       ├─ 需过滤 → 设置 formatter strip 标记 / 直接剥离 memory 中的媒体块
  │       └─ ___
  │
  ├─ [3] try: msg = await super()._reasoning(tool_choice)
  │       │     ↓
  │       │   ToolGuardMixin._reasoning (透传)
  │       │     ↓
  │       │   ReActAgent._reasoning (实际模型调用)
  │       │     ├─ formatter.format(msgs) → 拼装请求
  │       │     ├─ model.__call__(prompt, tools, tool_choice)
  │       │     ├─ 构造 assistant Msg
  │       │     ├─ await self.print(msg, True) → 流式输出
  │       │     ├─ await self.memory.add(msg) → 加入记忆
  │       │     └─ 处理用户中断（填充 fake tool_result）
  │       │
  │       └─ except (bad-request / media error):
  │           ├─ 被动重试：剥离媒体块后再次调用模型
  │           ├─ 记录 rejects_media 到 capability cache
  │           └─ 日志警告（多模态标记可能不准）
  │
  ├─ [4] _filter_plan_tools(msg, nb)
  │       └─ 预锁定 _plan_awaiting_user_confirm（防并行竞争）
  │
  └─ [5] await _auto_continue_if_text_only(msg, tool_choice)
          └─ 纯文本时提示模型继续使用工具（最多额外 2 轮）
```

### 5.2 媒体过滤双层策略

| 层级 | 时机 | 触发条件 | 动作 |
|---|---|---|---|
| **主动** | 模型调用前 | 模型不支持多模态 / cache 标记 rejects_media | 剥离 memory 中所有 `{type: image/audio/video}` 块 |
| **被动** | 模型调用失败后 | 400 错误 / 媒体相关关键词匹配 | 再次剥离后重试，缓存标记 |

### 5.3 Auto-continue 机制（L858-928）

当 `auto_continue_on_text_only` 开启时，模型返回纯文本（无 tool_use）：
```
_auto_continue_if_text_only(msg, tool_choice):
  while extra < 2:
      └─ 注入 language-matched hint + 尾部摘要
      └─ await super()._reasoning(tool_choice)
          ├─ 出现 tool_use → 采纳新消息
          └─ 仍纯文本 → 保留原消息，退出循环
```

**关键设计：** hint 消息标记为 `_MemoryMark.HINT`，下一轮 `_reasoning` 前自动清除，不污染记忆。

### 5.4 Plan 文本限制门控

```
当 agent 刚执行了 create_plan / revise_current_plan：
  └─ nb._plan_text_only_after_mutation = True
  下一轮 _reasoning：
  └─ tool_choice = "none"（禁止模型产生并行工具调用）
  一轮后自动恢复。
```

---

## 六、`_acting`— 工具执行层（L761-815）

### 6.1 调用链

```
QwenPawAgent._acting(tool_call)
  │
  ├─ [1] Plan 工具 JSON 参数修复
  │       └─ _fix_stringified_json_args(tool_call)
  │           └─ 对 subtask / subtasks 字段：json.dumps 字符串 → 解析为对象
  │
  ├─ [2] 设置 _plan_awaiting_user_confirm = True
  │       └─ Plan 变更工具调用前预锁定，防止 asyncio.gather 并行竞争
  │
  ├─ [3] check_plan_tool_gate(nb, tool_name)
  │       ├─ Plan 门未开启 → 放行
  │       └─ Plan 门开启且非 Plan 工具 → 返回错误消息
  │           └─ 构造 ToolResultBlock 错误消息，加入记忆，返回 None
  │
  ├─ [4] result = await super()._acting(tool_call)
  │       │     ↓
  │       │   ToolGuardMixin._acting
  │       │     ├─ _decide_guard_action(tool_call)
  │       │     │   ├─ denied → auto_denied
  │       │     │   ├─ STRICT → 全部需审批
  │       │     │   ├─ SMART → INFO/LOW 自动放行，Medium+ 需审批
  │       │     │   ├─ AUTO → 仅 guarded tools
  │       │     │   └─ OFF → 绕过
  │       │     │
  │       │     └─ 执行决策：
  │       │         ├─ auto_denied → _acting_auto_denied() → ToolResultBlock("禁止")
  │       │         ├─ needs_approval → _acting_with_approval()
  │       │         │   └─ 阻塞等待 300s → APPROVED/DENIED/TIMEOUT
  │       │         └─ auto_allowed → super()._acting(tool_call)
  │       │                              ↓
  │       │                           ReActAgent._acting
  │       │                             └─ toolkit.call_tool_function(tool_call)
  │       │                                 ├─ builtin tools
  │       │                                 ├─ plugin tools
  │       │                                 ├─ skill tools
  │       │                                 └─ MCP clients
  │       │
  │       └─ 工具结果加入 memory
  │
  └─ [5] Plan 后置处理
          └─ 设置 nb._plan_text_only_after_mutation = True
```

### 6.2 Token 流（`ReActAgent._acting` 内部时序）

```
toolkit.call_tool_function(tool_call)
  │
  ├─ 创建空的 ToolResultBlock msg
  │
  ├─ async for chunk in tool_res:
  │   ├─ 更新 msg.content
  │   └─ await self.print(msg, chunk.is_last)
  │
  └─ await self.memory.add(msg)
```

---

## 七、`_summarizing`— 兜底回复（L1106-1208）

当循环耗尽 `max_iters` 仍未产生文本回复时调用。

```
_summarizing()
  │
  ├─ [1] 主动媒体过滤（同 _reasoning 模式）
  │
  ├─ [2] self._in_summarizing = True
  │
  ├─ [3] try: msg = await super()._summarizing()
  │       │     ↓
  │       │   ReActAgent._summarizing
  │       │     ├─ 注入 hint: "无法在 max_iters 内生成回复"
  │       │     └─ model.__call__(hint_prompt)
  │       │
  │       └─ except (media error): 剥离重试
  │
  ├─ [4] finally: self._in_summarizing = False
  │
  └─ [5] return _strip_tool_use_from_msg(msg)
          └─ 移除模型产生的幽灵 tool_use blocks
          └─ 追加 round-end notice
```

**幽灵工具调用问题：** 某些模型（如 kimi-k2.5）在无工具模式下仍返回 `tool_use` block，这些 block 永远不会被执行，必须剥离。

---

## 八、`print`— 流式输出过滤（L1210-1255）

仅当 `_in_summarizing = True` 时生效：

```
print(msg, last, speech)
  │
  ├─ 非 summarizing 模式 → super().print()（透传）
  │
  └─ summarizing 模式：
      ├─ 过滤 content 中的 tool_use blocks
      ├─ 过滤后空内容且非末帧 → 跳过（避免 UI 闪烁）
      └─ 末帧 → 追加 round-end notice
```

---

## 九、MCP 客户端生命周期（L534-716）

### 9.1 注册流程

```
register_mcp_clients(namesake_strategy)
  │
  └─ for each mcp_client in self._mcp_clients:
      └─ toolkit.register_mcp_client(client, namesake_strategy, timeout)
          ├─ 成功 → 注册工具
          ├─ ClosedResourceError / CancelledError → 尝试恢复
          └─ 其他异常 → 跳过（不阻断启动）
```

### 9.2 会话恢复逻辑

```
_recover_mcp_client(client)
  │
  ├─ [_reconnect_mcp_client(client)]
  │   ├─ client.close()（忽略异常）
  │   └─ client.connect()（带 60s 超时）
  │
  ├─ 重连失败 → [重建客户端]
  │   └─ _rebuild_mcp_client(client)
  │       ├─ StdIOStatefulClient（stdio 传输）
  │       └─ HttpStatefulClient（http 传输）
  │
  └─ 重建后重连成功？
      ├─ 是 → _reuse_shared_client_reference()
      │       └─ 通过 __dict__.update 保持共享引用稳定
      └─ 否 → 返回 None
```

### 9.3 取消传播控制

`_should_propagate_cancelled_error()`（L632-648）：区分 MCP 内部取消和任务取消。
- Python >= 3.11: 通过 `task.cancelling()` 检查取消深度
- Python < 3.11: 保守传播所有 CancelledError

---

## 十、工具对象关系

```
agent_config (AgentProfileConfig)
  ├── running.max_iters, auto_continue_on_text_only, memory_compact_threshold
  ├── tools.builtin_tools → { name: { enabled, async_execution } }
  ├── approval_level (STRICT/SMART/AUTO/OFF)
  └── language

request_context (dict)
  ├── session_id, user_id, channel, agent_id
  └── 影响 skill 解析和审批路由

workspace_dir (Path)
  ├── AGENTS.md / SOUL.md / PROFILE.md → system prompt
  ├── skills/ → registered skill directories
  └── BOOTSTRAP.md → first-interaction guidance
```

---

## 十一、关键设计点总结

| 设计 | 说明 |
|---|---|
| **MRO 拦截链** | `ToolGuardMixin` 在 MRO 中位于 `QwenPawAgent` 和 `ReActAgent` 之间，安全拦截与核心逻辑透明融合 |
| **双层媒体过滤** | 主动（调用前检查能力）+ 被动（调用失败后重试），配合 capability cache 避免重复失败 |
| **Plan 门控三件套** | `_plan_awaiting_user_confirm`（预锁定防并行）、`_plan_text_only_after_mutation`（限制后续推理）、`check_plan_tool_gate`（拒绝非 plan 工具） |
| **Auto-continue** | 纯文本回复时注入 HINT 消息触发最多 2 轮额外推理，HINT 消息自动清除不污染记忆 |
| **审批对 Agent 透明** | 拒绝/超时产生 ToolResultBlock，Agent 无法区分是真实工具返回还是拒绝消息 |
| **幽灵 tool_use 过滤** | `_summarizing` 和 `print` 中去除模型在无工具模式下产生的无用 tool_use block |
| **MCP 会话恢复** | ClosedResourceError 时自动重连/重建，通过 `__dict__.update` 保持共享引用稳定 |
| **插件工具安全门控** | 不在硬编码列表中的工具需显式配置 enabled=true 才能注册 |
| **记忆工具注册** | `memory_manager.list_memory_tools()` 在构造后注册，不影响构造函数中的 system prompt 构建 |
| **环境变量每轮注入** | `apply_skill_config_env_overrides` 是上下文管理器，仅在 `super().reply()` 期间生效 |
