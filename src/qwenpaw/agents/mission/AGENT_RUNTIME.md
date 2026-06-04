# Agent Runtime 与 Mission Mode 交互流程

## 概述

QwenPaw 的 Agent Runtime 是一个基于 **懒加载 + HTTP API + SSE 流式通信** 的多 Agent 系统。Mission Mode 中的 Worker 并非独立的 Agent 实例，而是**同一个 Agent 的不同会话（session）**，通过 HTTP API 以新 session ID 调用。

---

## 1. Agent 定义

### 1.1 配置层次

Agent 配置分为两层：

**根配置 `config.json`** — 只存储引用：
```python
class AgentProfileRef(BaseModel):
    id: str              # 唯一 agent ID
    workspace_dir: str   # 工作空间目录路径
    enabled: bool = True # 是否启用（控制实例加载）
```

**工作空间配置 `workspace/agent.json`** — 完整配置：
```python
class AgentProfileConfig(BaseModel):
    id: str
    name: str                          # 人类可读名称
    description: str
    workspace_dir: str
    channels: Optional[ChannelConfig]  # 通信渠道（微信、console 等）
    mcp: Optional[MCPConfig]           # MCP 工具客户端
    running: AgentsRunningConfig       # 运行时配置
    llm_routing: AgentsLLMRoutingConfig # LLM 路由
    active_model: Optional[ModelSlotConfig] # 活跃模型
    language: str                      # 语言设置
    approval_level: str                # 工具执行安全级别
    system_prompt_files: List[str]     # 系统提示文件
    tools: Optional[ToolsConfig]       # 工具配置
    security: Optional[SecurityConfig] # 安全配置
    plan: PlanConfig                   # Plan 模式配置
```

### 1.2 配置加载

```python
# config/config.py
def load_agent_config(agent_id: str) -> AgentProfileConfig:
    """加载 agent 的完整配置"""
    config = load_config()
    if agent_id not in config.agents.profiles:
        raise ConfigurationException(...)
    agent_ref = config.agents.profiles[agent_id]
    # 从 workspace/agent.json 加载完整配置
    ...
```

---

## 2. Agent 启动流程

### 2.1 MultiAgentManager — 全局管理器

```python
class MultiAgentManager:
    """管理多个 Agent Workspace 的生命周期"""
    agents: Dict[str, Workspace] = {}  # 已加载的 agent
    _lock = asyncio.Lock()              # 异步锁
    _pending_starts: Dict[str, asyncio.Event] = {}  # 正在启动的 agent
```

### 2.2 懒加载机制

Agent **不会预启动**，在第一次被请求时才创建：

```python
async def get_agent(self, agent_id: str) -> Workspace:
    # 快速路径：已加载（无锁）
    if agent_id in self.agents:
        return self.agents[agent_id]

    async with self._lock:
        # 再次检查（可能其他协程已加载）
        if agent_id in self.agents:
            return self.agents[agent_id]

        if agent_id in self._pending_starts:
            # 另一个协程正在启动此 agent，等待
            event = self._pending_starts[agent_id]
        else:
            # 我们是第一个调用者 — 验证配置并声明启动权
            config = load_config()
            if agent_id not in config.agents.profiles:
                raise ConfigurationException(...)
            event = asyncio.Event()
            self._pending_starts[agent_id] = event
            should_start = True

    if not should_start:
        await event.wait()  # 等待其他协程完成启动
        return self.agents[agent_id]

    # 创建 Workspace（在锁外，允许并行启动）
    instance = Workspace(
        agent_id=agent_id,
        workspace_dir=agent_ref.workspace_dir,
    )
    await instance.start()
    instance.set_manager(self)

    async with self._lock:
        self.agents[agent_id] = instance
    return instance
```

**关键设计**：锁只在字典检查/更新时持有，不持有整个启动过程，允许多个 agent 并行初始化。

### 2.3 Workspace 启动

`Workspace` 封装了一个完整的 Agent 运行时：

```python
class Workspace:
    """单个 Agent 的完整运行时"""
    runner: AgentRunner           # 请求处理引擎
    memory_manager               # 对话记忆
    context_manager              # 上下文管理
    mcp_manager: MCPClientManager # MCP 工具客户端
    chat_manager                 # 聊天管理
    channel_manager              # 通信渠道管理
    cron_manager: CronManager    # 定时任务
    task_tracker: TaskTracker    # 后台任务跟踪
```

`workspace.start()` 启动所有服务：

```python
async def start(self):
    # 1. 加载配置
    self._config = load_agent_config(self.agent_id)

    # 2. 初始化技能池
    ensure_skill_pool_initialized()

    # 3. 数据迁移（legacy weixin -> wechat）
    self._migrate_legacy_weixin_data()

    # 4. 按优先级启动所有服务
    await self._service_manager.start_all()
```

服务按优先级启动：
- **Priority 10**: Runner（请求处理）
- **Priority 20**: 核心服务（Memory、Context、MCP、Chat）— 并发启动
- **Priority 25**: Runner.start()
- **Priority 30**: ChannelManager（通信渠道）
- **Priority 40**: CronManager（定时任务）
- **Priority 50-51**: Config Watchers（配置文件监听）

---

## 3. Agent 通信机制

### 3.1 API 路由

所有 Agent API 都在 `/agents/{agentId}/` 路径下：

```
/api/agents/{agentId}/console/chat     # 控制台对话
/api/agents/{agentId}/chats/*           # 聊天管理
/api/agents/{agentId}/config/*          # 配置管理
/api/agents/{agentId}/mcp/*             # MCP 工具
/api/agents/{agentId}/skills/*          # 技能管理
/api/agents/{agentId}/tools/*           # 工具管理
/api/agents/{agentId}/workspace/*       # 工作空间
```

### 3.2 AgentContextMiddleware — Agent 路由

```python
class AgentContextMiddleware(BaseHTTPMiddleware):
    """从请求中提取 agentId 并注入上下文"""
    async def dispatch(self, request, call_next):
        # Priority 1: 从路径提取 /api/agents/{agentId}/...
        agent_id = path_parts[3]

        # Priority 2: 从 X-Agent-Id header 提取
        if not agent_id:
            agent_id = request.headers.get("X-Agent-Id")

        # 设置 agent_id 到上下文变量
        if agent_id:
            set_current_agent_id(agent_id)

        # 提取 root_session_id 用于跨会话审批路由
        root_session_id = request.headers.get("X-Root-Session-Id")
        ...
```

### 3.3 请求处理流程

```
HTTP Request
    ↓
AgentContextMiddleware (提取 agentId)
    ↓
get_agent_for_request() → MultiAgentManager.get_agent(agent_id)
    ↓
Workspace.runner.query_handler(msgs, request)
    ↓
AgentRunner 处理请求（命令/技能/LLM 调用）
    ↓
SSE 流式返回响应
```

### 3.4 Agent 间通信工具

`agent_management.py` 提供三个通信工具：

| 工具 | API 端点 | 阻塞？ | 用途 |
|------|----------|--------|------|
| `chat_with_agent(to_agent, text)` | `POST /agent/process` (SSE) | 是 | 前台同步等待 |
| `submit_to_agent(to_agent, text)` | `POST /agent/process/task` | 否 | 提交后台任务 |
| `check_agent_task(task_id)` | `GET /agent/process/task/{id}` | 否 | 轮询任务状态 |

**消息构建**：
```python
def build_agent_chat_request(to_agent, text, session_id, from_agent, root_session_id):
    request_payload = {
        "session_id": final_session_id,
        "input": [{"role": "user", "content": [{"type": "text", text}]}],
        "request_context": {"root_agent_id": caller_agent_id},
    }
    if root_session_id:
        request_payload["root_session_id"] = root_session_id
    return final_session_id, request_payload, ...
```

**Session ID 生成**：
```python
def generate_unique_session_id(from_agent, to_agent):
    timestamp = int(time.time() * 1000)
    uuid_short = str(uuid4())[:8]
    return f"{from_agent}:to:{to_agent}:{timestamp}:{uuid_short}"
```

---

## 4. Mission Mode 中的 Worker 机制

### 4.1 Worker 不是独立 Agent

**关键发现**：Mission Mode 中的 Worker **不是**动态创建的独立 Agent 实例，而是**同一个 Agent 的不同会话（session）**。

当 Master 调用：
```python
submit_to_agent(to_agent="default", text=worker_prompt)
```

这实际上：
1. 生成新的 session ID：`default:to:default:1719456789000:a1b2c3d4`
2. 发送 HTTP 请求到同一个 Agent 的 API：`POST /api/agents/default/agent/process/task`
3. Worker 的 "Agent" 就是同一个 `default` Agent，只是以全新 session 运行
4. Worker 没有记忆（新 session），所有上下文都在 prompt 中

### 4.2 Worker 执行流程

```
Master Agent (default, session: master-session)
    │
    ├── submit_to_agent(to_agent="default", text=worker_prompt)
    │   ↓
    │   HTTP POST /api/agents/default/agent/process/task
    │   {session_id: "default:to:default:...", input: [{...}]}
    │   ↓
    │   返回 task_id
    │
    ├── check_agent_task(task_id)  # 轮询状态
    │   ↓
    │   GET /api/agents/default/agent/process/task/{task_id}
    │   {status: "running"} → {status: "finished"}
    │
    └── 读取 prd.json / progress.txt  # 验证结果
```

### 4.3 Worker 的 "身份"

Worker 实际上是：
- **同一个 Agent 进程**（同一个 Workspace 实例）
- **新的 Session**（无历史对话）
- **不同的 System Prompt**（Worker Instructions + Story JSON）
- **相同的工具集**（但 Phase 2 限制了实现工具）
- **相同的工作空间目录**（`loop_dir`）

### 4.4 为什么这样设计

1. **资源效率**：不需要为每个 worker 创建新的 Agent 进程
2. **一致性**：所有 worker 使用相同的配置、模型、工具
3. **简化管理**：无需动态注册/注销 Agent
4. **隔离性**：通过 session ID 实现会话隔离，worker 之间互不干扰

### 4.5 与"真正多 Agent"的区别

| 方面 | QwenPaw Mission Mode | 真正多 Agent |
|------|---------------------|-------------|
| Agent 实例 | 单个实例，多 session | 每个 worker 独立实例 |
| 进程模型 | 共享进程 | 可独立进程 |
| 资源占用 | 低（共享） | 高（每个独立） |
| 隔离级别 | Session 级 | 进程/实例级 |
| 动态性 | 固定 Agent，动态 session | 动态创建 Agent |

---

## 5. 完整交互流程图

### 5.1 Agent 启动流程

```
                    config.json
                        │
                        ▼
              ┌─────────────────┐
              │ MultiAgentManager│
              │   (懒加载)       │
              └────────┬────────┘
                       │ 第一次请求
                       ▼
              ┌─────────────────┐
              │   Workspace     │
              │ ┌─────────────┐ │
              │ │   Runner    │ │ ← 请求处理
              │ │   Memory    │ │ ← 对话记忆
              │ │   MCP       │ │ ← 工具客户端
              │ │   Channels  │ │ ← 通信渠道
              │ │   Cron      │ │ ← 定时任务
              │ └─────────────┘ │
              └────────┬────────┘
                       │ start()
                       ▼
              ┌─────────────────┐
              │  Agent Ready    │
              │  HTTP API 就绪  │
              └─────────────────┘
```

### 5.2 Mission Mode 执行流程

```
用户: /mission 实现用户认证系统
        │
        ▼
┌─────────────────────────────────────────┐
│ Master Agent (default)                  │
│                                         │
│ Phase 1: PRD 生成                       │
│ ├── 探索代码库                          │
│ ├── 生成 prd.json                       │
│ └── 等待用户确认                        │
│                                         │
│ Phase 2: 执行循环                       │
│ ├── 读取 prd.json (未完成 stories)      │
│ ├── submit_to_agent(to_agent="default") │
│ │   ↓                                  │
│ │   HTTP POST /agent/process/task      │
│ │   {session_id: "worker-1", ...}      │
│ │   ↓                                  │
│ │   Worker 1 执行 (新 session)          │
│ │   写入 prd.json, progress.txt         │
│ │   ↓                                  │
│ │   返回 task_id                        │
│ │                                       │
│ ├── submit_to_agent(to_agent="default") │
│ │   ↓                                  │
│ │   Worker 2 执行 (新 session)          │
│ │                                       │
│ ├── check_agent_task(task_id_1)         │
│ │   等待完成...                         │
│ │                                       │
│ ├── 验证 prd.json                       │
│ │   全部 passes: true? ──→ 完成        │
│ │   否 ──→ 继续循环                     │
│ └──                                     │
└─────────────────────────────────────────┘
```

---

## 6. 关键代码入口

| 组件 | 文件 | 作用 |
|------|------|------|
| Agent 配置 | `config/config.py` | `AgentProfileRef`, `AgentProfileConfig` |
| 全局管理 | `app/multi_agent_manager.py` | 懒加载、生命周期管理 |
| 工作空间 | `app/workspace/workspace.py` | 封装完整运行时 |
| 请求处理 | `app/runner/runner.py` | `AgentRunner.query_handler()` |
| Agent 路由 | `app/routers/agent_scoped.py` | `/agents/{agentId}/` 路由 |
| Agent 上下文 | `app/agent_context.py` | agent_id 注入 |
| 通信工具 | `agents/tools/agent_management.py` | `submit_to_agent`, `check_agent_task` |
| Mission 调度 | `agents/mission/handler.py` | `/mission` 命令处理 |
| Mission 执行 | `agents/mission/mission_runner.py` | Phase 1/2 循环 |
| Mission 提示 | `agents/mission/prompts.py` | Master/Worker/Verifier prompt |
| Mission 状态 | `agents/mission/state.py` | 文件读写、git 检测 |

---

## 7. 总结

1. **Agent 是懒加载的**：定义好配置后，第一次通信时才创建 Workspace 并启动全部服务
2. **通信是 HTTP API**：所有 Agent 交互都经过 FastAPI 路由层，通过 `X-Agent-Id` header 或 URL 路径路由
3. **Worker 是同 Agent 的新 Session**：Mission Mode 中的 Worker 不是独立 Agent，而是同一个 Agent 以新 session ID 运行，通过 prompt 注入上下文
4. **文件系统是协调核心**：Master、Worker、Verifier 通过读写 `loop_dir` 下的文件（prd.json、progress.txt）实现状态同步
5. **Session 隔离**：每次调度生成唯一 session ID，确保 worker 之间互不干扰
