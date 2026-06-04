# Mission Mode 架构文档

## 概述

Mission Mode 是 QwenPaw 的自主迭代执行引擎，将复杂任务自动分解、调度、验证直至完成。

## 两阶段生命周期

### Phase 1 — PRD 生成

1. 用户执行 `/mission <任务描述>`，`handler.py` 创建 mission 目录、初始化状态文件、检测 git 环境
2. 通过 `prompts.py` 的 `build_master_prompt()` 构建 master prompt，注入到主 agent
3. 主 agent **可使用所有工具**（探索代码库、搜索文件），将任务分解为结构化的 `prd.json`（User Stories）
4. PRD 有 schema 校验（`validate_prd()`），不合格会自动注入修正 prompt，最多重试 2 次
5. Agent 向用户展示 PRD 并等待确认
6. 用户确认后，agent 将 `loop_config.json` 的 `current_phase` 设为 `execution_confirmed`

### Phase 2 — 执行循环

1. **工具限制**：调用 `set_phase2_tool_restrictions()` 将 `edit_file`、`browser_use`、`desktop_screenshot` 等实现工具移入独立 group 并禁用。Master agent 只能读文件和调度 worker
2. **迭代循环**（最多 `max_iterations` 轮）：
   - 读取 `prd.json`，按 priority 分组，取最低 priority 的 story 批次
   - 为每个 story 构造 worker prompt，通过 `submit_to_agent` **并行调度** worker
   - Worker 是全新 session（无记忆），在 `loop_dir` 中独立工作
   - Master 通过 `check_agent_task` 轮询 worker 状态
   - 所有 worker 完成后，调度 **verifier**（对抗性验证 agent）对每个 story 独立验证
   - Verifier 输出 `VERDICT: PASS/FAIL/PARTIAL`，master 据此更新 `prd.json` 的 `passes` 字段
   - FAIL 的 story 会带上错误上下文重新派发 worker（最多重试 3 次）
3. **终止条件**：所有 story 的 `passes: true`，或达到最大迭代次数

## 状态文件布局

```
{workspace_dir}/missions/{mission-YYYYMMDD-HHMMSS}/
├── loop_config.json   # 环境元数据（git、路径、阶段、session_id）
├── prd.json           # 任务清单（master 更新 passes 字段）
├── progress.txt       # 追加式迭代日志 + Codebase Patterns
└── task.md            # 原始任务描述（只读）
```

## Master-Worker 通信机制

基于 **HTTP API + SSE（Server-Sent Events）** 的异步任务模式。

### 通信工具

| 工具 | API 端点 | 阻塞？ | 用途 |
|------|----------|--------|------|
| `submit_to_agent(to_agent, text)` | `POST /agent/process/task` | 否 | 提交后台任务，返回 task_id |
| `check_agent_task(task_id)` | `GET /agent/process/task/{id}` | 否 | 轮询任务状态 |
| `chat_with_agent(to_agent, text)` | `POST /agent/process` (SSE) | 是 | 前台同步等待 |

Mission Mode 主要使用 `submit_to_agent` + `check_agent_task` 组合（异步轮询）。

### 通信流程

```
Master (控制器)                          Server (API)                          Worker
     |                                      |                                   |
     |-- POST /agent/process/task --------->|                                   |
     |   {session_id, input, agent_id}      |-- 启动 worker session ---------->|
     |<-- {task_id, session_id} ------------|                                   |
     |                                      |                                   |
     |-- (轮询) GET /agent/process/task/{id} ->|                               |
     |<-- {status: "running"} -------------|  <-- worker 正在执行... -->       |
     |                                      |                                   |
     |-- GET /agent/process/task/{id} ----->|                                   |
     |<-- {status: "finished"} ------------|<-- worker 完成，写prd.json       |
     |   {result: {output: [{content}]}}   |                                   |
```

### 关键实现细节

**Session 隔离**（`agent_management.py:77-81`）：每次调度生成唯一 session ID `{from}:to:{to}:{timestamp}:{uuid}`，确保 worker 之间互不干扰。

**消息构建**（`agent_management.py:192-235`）：消息通过 `X-Agent-Id` header 路由到目标 agent，payload 包含 `session_id`、`input`（role=user）、`request_context`（root_agent_id）。

**身份前缀**（`agent_management.py:110-124`）：消息自动加 `[Agent {id} requesting]` 前缀，让 worker 知道来源。

### 数据流

- **写通道**（Master → Worker）：Master 通过 `submit_to_agent` 发送包含完整 prompt 的消息（story JSON + Worker Instructions + Codebase Patterns）
- **读通道**（Worker → Master）：Worker 写入 `loop_dir/prd.json`（设置 `passes`）和 `loop_dir/progress.txt`；Master 通过 `check_agent_task` 获取 worker 文本输出，同时读取文件验证进度
- **共享状态文件**是核心协调机制 — 所有 agent 通过文件系统实现状态同步

## 文件系统同步竞争分析

### 写入函数无保护

`state.py` 中的 `write_loop_config`、`write_prd_json` 等都是裸的 `Path.write_text()` — **无原子写入（temp+rename）、无文件锁**。

### 逐文件竞争评估

| 文件 | 写入者 | 读者 | 竞争风险 |
|------|--------|------|----------|
| `prd.json` | Master only | Master + Worker | **安全** — 设计上只有 master 写，worker 只读 |
| `loop_config.json` | Master only | Master | **安全** — 单写者 |
| `progress.txt` | Worker(s) | Master + Worker | **有风险** — 多 worker 并发 append 可能交错 |
| `task.md` | 初始化一次 | Master + Worker | **安全** — 只读 |

### progress.txt 并发 append 问题

多个 worker 并行时同时 append 到 progress.txt：
- 实际由 LLM agent 通过 `write_file` 或 shell `echo >>` 执行 append
- **可能结果**：交错写入导致内容混乱，但不会丢失（OS 级 `write()` 对小块通常原子）
- **实际影响有限**：progress.txt 是日志性质，偶尔交错不影响核心逻辑（prd.json 才是 source of truth）

### 为什么竞争风险整体可控

1. **架构隔离**：Phase 2 禁用了 master 的实现工具，master 只通过 API 调度 worker，不直接写代码文件
2. **读写分离**：prd.json 是核心状态，只有 master 写 `passes` 字段；worker 只读 prd.json
3. **串行化调度**：同一 priority batch 的 worker 虽然并行，但 master 在**所有 worker 完成后**才读取 prd.json 做判断，不存在读到半写状态
4. **Worker 无状态**：每个 worker 是全新 session，不共享内存，唯一共享介质是文件系统
5. **无 write-write conflict**：每个 worker 负责不同 story，不写同一个字段

### 可选改进方向

- `write_prd_json()` 可改为 atomic write（写临时文件 + `os.rename`）
- `progress.txt` 可加文件锁或改为每个 worker 独立日志文件再合并
- 当前实现在实践中足够可靠，因为 LLM agent 的执行速度远慢于文件 I/O
