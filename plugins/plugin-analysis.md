# QwenPaw 插件系统内部实现原理分析

## 一、插件引擎核心

**代码目录：** `src/qwenpaw/plugins/`

### 1.1 架构分层

| 层 | 文件 | 职责 |
|---|---|---|
| 数据模型 | `architecture.py` | `PluginType` 枚举、`PluginManifest`、`PluginRecord`、`PluginEntryPoints` |
| 开发者接口 | `api.py` | `PluginApi` 类，插件开发者的唯一 API 入口 |
| 生命周期 | `loader.py` | `PluginLoader` 类，插件的发现/加载/卸载 |
| 注册中心 | `registry.py` | `PluginRegistry` 单例，管理所有注册信息 |
| 运行时工具 | `runtime.py` | `RuntimeHelpers`，提供给插件的日志和 Provider 查询能力 |
| 下载目录 | `download_catalog.py` | 从 CDN 获取官方插件市场元数据 |

### 1.2 数据模型 (`architecture.py`)

**`PluginType` 枚举** — 六种规范类型：
- `TOOL`：注册一个或多个 Agent 工具函数（LLM 可调用）
- `PROVIDER`：注册自定义 LLM Provider
- `HOOK`：应用启动/关闭时运行代码
- `COMMAND`：注册 `/slash` 控制命令
- `FRONTEND`：提供前端 JS 包（UI 动态加载）
- `GENERAL`：不属于以上任何类型的回退

**`PluginManifest`** — 从 `plugin.json` 解析的核心数据结构：
- `id`、`name`、`version`、`description`、`author`
- `entry: PluginEntryPoints`（`frontend` + `backend` 路径）
- `dependencies: List[str]`
- `min_version`：最低 QwenPaw 版本
- `meta: Dict`：插件特定元数据（工具定义、配置字段等）
- `plugin_type: PluginType`：显式指定或从 `meta` 自动推断

**关键方法 `from_dict()`**：解析 `plugin.json` 字典，优先使用 `"type"` 字段；对旧版本插件（无 `type` 字段）通过 `_type_from_meta()` 反向推断。

**`PluginRecord`** — 运行时包装：
```python
PluginRecord(manifest, source_path, enabled=True, instance)
```

### 1.3 插件清单 (`plugin.json`) 结构

必备字段：`id`、`name`、`version`

可选字段：`type`、`description`、`author`、`entry`（`frontend`/`backend`）、`dependencies`、`min_version`、`meta`

`meta` 是核心扩展点：
- 旧格式：`tool_name`（字符串）
- 新格式：`tools`（数组，每个元素含 `name`、`description`、`icon`、`requires_config`、`config_fields`）
- `config_fields` 数组定义 UI 配置字段：`name`、`label`、`type`（text/password/number/select）、`required`、`placeholder`、`default`、`options`、`help`

### 1.4 生命周期：发现 → 加载 → 运行 → 卸载

#### 发现 (`PluginLoader.discover_plugins()`)
```
遍历 plugin_dirs 中的每个子目录
  └─ 检查 plugin.json 是否存在
      └─ 解析 PluginManifest
          └─ 返回 [(manifest, plugin_dir), ...]
```

#### 加载 (`PluginLoader.load_plugin()`)
```
输入: manifest, source_path, config

1. 检查是否重复加载
2. 解析 backend 入口（默认 plugin.py）
3. 若 backend 存在:
   a. 通过 importlib.util.spec_from_file_location() 动态导入
   b. 设置 submodule_search_locations 为插件目录（启用相对导入）
   c. 创建唯一模块名 plugin_{plugin_id}
   d. 执行模块，获取 plugin 对象
   e. 创建 PluginApi(plugin_id, config, manifest_dict)
   f. 注册 PluginManifest 到 Registry
   g. 调用 plugin.register(api)（支持同步和异步）
4. 创建 PluginRecord(manifest, source_path, enabled=True, instance)
```

**关键设计：**
- `submodule_search_locations` 参数使插件内可相对导入，无需修改 `sys.path`
- 模块名使用 `plugin_{plugin_id}` 格式避免命名冲突
- `register()` 同时支持 sync 和 async（用 `inspect.iscoroutine()` 检测）

#### 从路径加载 (`PluginLoader.load_plugin_from_path()`)
```
1. 解析 source_path，读取 plugin.json
2. 路径穿越防护: 检查 target_dir.is_relative_to(install_dir)
3. 复制插件目录到 install_dir（如尚未在目标目录）
4. 安装 requirements.txt 依赖（asyncio.to_thread 避免阻塞事件循环）
5. 重新读取新位置的 manifest，调用 load_plugin()
```

#### 依赖安装 (`_install_requirements()`)
```
Attempt 1: python -m pip install -r requirements.txt
  └─ 失败且错误为 "No module named pip"
      └─ Attempt 2: uv pip install --python <sys.executable> -r <requirements>
```
支持 pip 和 uv 管理的虚拟环境。超时 300 秒。

#### 卸载 (`PluginLoader.unload_plugin()`)
```
1. 执行该插件注册的所有 shutdown hooks
2. 从 sys.modules 移除 plugin_{plugin_id} 及其所有子模块
3. 调用 registry.unregister_plugin(plugin_id)
4. 从 _loaded_plugins 字典移除
5. _cleanup_plugin_tools(): 从 qwenpaw.agents.tools 模块中删除工具函数
6. 可选删除磁盘文件
```

### 1.5 注册中心 (`PluginRegistry`)

单例模式（`__new__` 实现）。

**注册类型：**

| 类型 | 存储 | 注册方法 | 查询方法 |
|---|---|---|---|
| Providers | `_providers: Dict[str, ProviderRegistration]` | `register_provider()` | `get_provider()` / `get_all_providers()` |
| 启动钩子 | `_startup_hooks: List[HookRegistration]` | `register_startup_hook()` | `get_startup_hooks()` |
| 关闭钩子 | `_shutdown_hooks: List[HookRegistration]` | `register_shutdown_hook()` | `get_shutdown_hooks()` |
| 控制命令 | `_control_commands: List[ControlCommandRegistration]` | `register_control_command()` | `get_control_commands()` |
| HTTP 路由 | `_http_router_registrations: List[HttpRouterRegistration]` | `register_http_router()` | `get_http_router_registrations()` |
| 插件清单 | `_plugin_manifests: Dict[str, Dict]` | `register_plugin_manifest()` | `get_plugin_manifest()` / `get_all_plugin_manifests()` |
| 运行时工具 | `_runtime_helpers` | `set_runtime_helpers()` | `get_runtime_helpers()` |

**HTTP 路由挂载 (`register_http_router()`)：**
1. 验证 prefix 格式（以 `/` 开头，不能为 `/` 本身，不能重复注册）
2. 在 FastAPI app 上挂载 APIRouter 到 `/api{prefix}`
3. 将新路由插入到控制台 SPA catch-all 路由**之前**（通过 `_mount_plugin_http_on_app()`），确保插件路由优先匹配
4. 清除 FastAPI 缓存的 OpenAPI schema

**工具配置 (`get_tool_config()` / `set_tool_config()`)：**
- 通过 `load_agent_config(agent_id)` 读写 agent 配置文件中的 `BuiltinToolConfig.config`
- 工具配置是**按 agent 隔离**的

**注销 (`unregister_plugin()`)：**
- 移除 HTTP 路由、清单、providers、启动钩子、关闭钩子、控制命令

### 1.6 开发者接口 (`PluginApi`)

`PluginApi` 实例由 `PluginLoader` 创建并传入插件的 `register()` 方法。

| 方法 | 用途 | 实现要点 |
|---|---|---|
| `register_provider(id, class, label, base_url, **metadata)` | 注册 LLM Provider | 合并 manifest meta，调用 registry |
| `register_startup_hook(name, callback, priority=100)` | 注册启动钩子 | 优先级排序（低值先执行） |
| `register_shutdown_hook(name, callback, priority=100)` | 注册关闭钩子 | 同上 |
| `register_http_router(router, *, prefix, tags)` | 暴露 REST 端点 | `/api{prefix}`，插入 SPA 路由之前 |
| `register_control_command(handler, priority_level=10)` | 注册控制命令 | 注册到全局 handler registry |
| `register_tool(name, func, description, icon, enabled=False)` | 注册工具函数 | 延迟到 startup hook 执行 |
| `get_tool_config(name, agent_id)` | 获取工具配置 | 委托 registry |
| `set_tool_config(name, agent_id, config)` | 保存工具配置 | 委托 registry |
| `runtime` (property) | 访问 RuntimeHelpers | 获取 provider 列表和日志能力 |

**`register_tool()` 内部实现：**
```python
def register_tool(tool_name, tool_func, ...):
    def _startup_register():
        # 1. setattr(tools_module, tool_name, tool_func) 注入到 tools 模块
        # 2. tools_module.__all__.append(tool_name)
        # 3. 创建 BuiltinToolConfig，写入 agent config（默认 disabled）
    self.register_startup_hook(
        hook_name=f"register_tool_{plugin_id}_{tool_name}",
        callback=_startup_register,
        priority=50,  # 默认中等优先级
    )
```
工具注册是**延迟的**——创建一个 startup hook 在应用完全初始化后执行。

**`get_tool_config()` 独立函数：** 工具函数可直接调用：
```python
from qwenpaw.plugins import get_tool_config
config = get_tool_config("my_tool")  # 内部通过 get_current_agent_id() 获取当前 agent
```

### 1.7 应用启动流程 (`src/qwenpaw/app/_app.py`)

```
Phase 1 (快速启动): 启动服务器，初始化 app.state
Phase 2 (后台启动: _background_startup()):
  1. Agent 启动
  2. 创建 PluginLoader([get_plugins_dir()])
  3. plugin_loader.registry.set_plugin_http_app(app) 绑定 FastAPI
  4. plugin_loader.load_all_plugins(configs) 发现并加载所有插件
  5. RuntimeHelpers(provider_manager) 设置到 registry
  6. 注册插件 providers 到 ProviderManager
  7. 注册控制命令到全局 handler + priority 排序的 command_registry
  8. 按优先级执行所有 startup hooks

关闭时:
  按优先级执行所有 shutdown hooks
```

### 1.8 安全措施

- **路径穿越防护**：`load_plugin_from_path()` 中使用 `target_dir.is_relative_to(install_dir)` 检查
- **Zip Slip 防护**：CLI 和 API 都验证 zip 成员不会解压到目标目录之外
- **后端入口验证**：安装前 import 检查是否包含 `Plugin` 类或 `plugin` 实例
- **静态文件服务**：路径穿越防护确保文件在插件源目录内
- **下载大小限制**：500 MB
- **单例注册中心**：`PluginRegistry.__new__` 确保全局唯一实例

---

## 二、工具插件 (Tool Plugins)

三个工具插件使用完全相同的结构模式。

### 2.1 共同模式

```
plugin.json          ← 元数据 + 工具定义
plugin_entry.py      ← Plugin 类 + plugin 实例导出
tool_impl.py         ← 工具函数实现
requirements.txt     ← 依赖
```

### 2.2 注册流程

```
plugin.py 模块加载时：
  └─ 导出 plugin = ClassName()
  
PluginLoader.load_plugin() 调用 plugin.register(api):
  └─ _load_tool_module() 通过 importlib 加载 *_tool.py
      └─ api.register_tool(name, func, ...) 创建延迟 startup hook (priority=50)
          
应用启动时执行 startup hooks:
  └─ _startup_register() 被调用
      ├─ setattr(qwenpaw.agents.tools, tool_name, tool_func)
      ├─ __all__.append(tool_name)
      └─ 在 agent config 中创建 BuiltinToolConfig (enabled=False)
```

### 2.3 GPT Image 2 (`plugins/tool/gpt-image2/`)

| 文件 | 作用 |
|---|---|
| `plugin.json` | ID: `gpt-image2-tool`, type: `tool`, 依赖 `httpx>=0.24.0` |
| `gpt_image2.py` | `GPTImage2ToolPlugin.register(api)` — 注册 `generate_image_gpt` + `edit_image_gpt` |
| `gpt_image2_tool.py` | 实际的工具函数实现 |

**工具函数：**
- `generate_image_gpt(prompt, size, quality)` → 调用 OpenAI `/images/generations`，保存为本地 PNG
- `edit_image_gpt(prompt, reference_images, size, quality)` → 调用 OpenAI `/images/edits`，支持多图
- 内部使用 `_process_image_url()` 将本地图片转为 base64 data URL 或保留 HTTP URL

**配置字段:** `api_key` (password), `endpoint` (text), `timeout` (number)

### 2.4 Qwen-Image (`plugins/tool/qwen-image/`)

| 文件 | 作用 |
|---|---|
| `plugin.json` | ID: `qwen-image-tool`, 依赖 `dashscope>=1.25.16` + `httpx>=0.24.0` |
| `qwen_image.py` | `QwenImageToolPlugin.register(api)` — 注册 `generate_image_qwen` + `edit_image_qwen` |
| `qwen_image_tool.py` | 实际实现 |

**工具函数：**
- `generate_image_qwen(prompt, size, n, negative_prompt, prompt_extend)` → 通过 `dashscope.MultiModalConversation.call()` 调用
- `edit_image_qwen(prompt, reference_images, ...)` → 同上，支持多图融合

**技术细节：**
- 通过 `asyncio.to_thread()` 将同步的 DashScope SDK 调用放到线程池
- 使用 `threading.Lock()` 保护 `dashscope.base_http_api_url` 全局变量
- 生成结果先下载到 `DEFAULT_MEDIA_DIR / "qwen_image"`，再以本地路径返回
- 模型选择通过 config 字段的 select 类型，可选用多种模型版本

### 2.5 Wan 2.7 视频生成 (`plugins/tool/wan27/`)

| 文件 | 作用 |
|---|---|
| `plugin.json` | ID: `wan27-tool`, 依赖 `dashscope>=1.25.16` + `httpx>=0.24.0` |
| `wan27.py` | `Wan27ToolPlugin.register(api)` — 注册 3 个工具 |
| `wan27_tool.py` | 实际实现 |

**工具函数：**
- `text_to_video_wan(prompt, resolution, ratio, duration, ...)` → `wan2.7-t2v-2026-04-25` 模型
- `image_to_video_wan(prompt, first_frame_url, last_frame_url, driving_audio_url, first_clip_url, ...)` → `wan2.7-i2v-2026-04-25` 模型，支持 4 种模式（first-frame / first-last-frame / audio-driven / video-continuation）
- `reference_to_video_wan(prompt, reference_images, reference_videos, ...)` → `wan2.7-r2v` 模型

**技术细节：**
- 通过 `dashscope.VideoSynthesis.call()` 调用
- 与 qwen-image 相同的线程安全模式（`_DASHSCOPE_LOCK` + `asyncio.to_thread`）
- 视频下载使用 httpx 流式下载 + 分块写入
- 支持 720P/1080P 分辨率，2-15 秒时长

---

## 三、Bundle 插件

### 3.1 CloudPaw (`plugins/bundle/cloudpaw/`)

**类型:** `general`，使用 `register_startup_hook()` + `register_shutdown_hook()`

**`plugin.json` 特点：**
- 同时有 `frontend` 和 `backend` 入口
- `meta.category = "cloud-deployment"`
- 无 Python 依赖（`"dependencies": []`）

**启动时完成的工作 (`cloudpaw_init`, priority=50)：**

```
1. _ensure_default_env_vars()
   └─ 在 envs.json 中预置 ALIBABA_CLOUD_* 环境变量

2. inject_interaction_module()
   └─ 注入合成模块到 sys.modules

3. _install_plugin_skills()
   └─ 复制 skills/ 目录到全局 skill pool
   └─ 更新 pool 的 skill.json 清单，标记 source: "plugin:cloudpaw"

4. ensure_builtin_agents()
   └─ 注册内置 Agent（编排/IaC/执行器/验证器）

5. setup_tool_and_prompt_hooks()
   └─ 注册 proposal_choice, manage_prd 等工具

6. setup_acp_auto_approve()
   └─ 配置 ACP 权限自动批准

7. setup_mission_hooks()
   └─ 配置任务模式钩子

8. mount_routers()
   └─ 挂载 API 路由（使用 FastAPI 直接挂载）

9. _init_a2a_manager()
   └─ 初始化 A2A 客户端管理器（Agent-to-Agent 协议）

10. _register_a2a_command()
    └─ 注册 /a2a 控制命令（A2AListCommandHandler）

11. _ensure_aliyun_cli()
    └─ 检测 aliyun CLI 是否存在，否则自动安装
    └─ macOS: bash script → Homebrew → tgz
    └─ Linux: bash script → tgz
    └─ Windows: PowerShell script

12. _patch_plugin_loader_unload()
    └─ Monkey-patch PluginLoader.unload_plugin
    └─ 卸载时调用 uninstall_agents() 清理内置 Agent
```

**关闭时：** 关闭 A2A 客户端管理器。

### 3.2 QwenPaw Pet (`plugins/bundle/qwenpaw-pet/`)

**类型:** 未显式设置，从 meta 推断为 `general`

**`plugin.json` 特点：**
- 同时有 `frontend` 和 `backend` 入口
- 依赖较多（httpx, fastapi, uvicorn, pillow, pyside6-essentials）

**插件目录结构：**
```
emitter.py          ← 事件发射 + 桌面进程管理
patch_approval.py   ← Patch ApprovalService
patch_runner.py     ← Patch AgentRunner
router.py           ← FastAPI router
dist/index.js       ← 前端 JS 包
frontend/           ← 前端源码
```

**启动时 (`qwenpaw_pet_startup`, priority=80)：**

```
1. patch_agent_runner()
   └─ 修改 AgentRunner 生命周期事件处理

2. patch_approval_service()
   └─ 修改 ApprovalService 审批事件处理

3. ensure_desktop_available()
   └─ 确保桌面宠物进程运行

4. emit_pet_event("qwenpaw.startup", ...)
   └─ 通过 HTTP 通知桌面宠物
```

**关闭时 (`qwenpaw_pet_shutdown`, priority=120)：**

```
1. emit_pet_event("qwenpaw.shutdown", ...)
2. stop_desktop()
3. restore_agent_runner()
4. restore_approval_service()
```

**HTTP 路由：** 挂载在 `/api/qwenpaw-pet/` 下，用于前后端通信。

**技术细节：**
- 启动时将插件目录加入 `sys.path`（第 14-15 行）以支持同目录模块导入
- Patch 失败不阻塞启动（try/except + logger.exception），保证 UI 不静默死掉

---

## 四、插件系统数据流图

```
用户/CLI/API                    PluginLoader                   PluginRegistry
    │                               │                               │
    ├── install(source) ────────────┤                               │
    │                               ├── 复制到 plugins/<id>/        │
    │                               ├── pip/uv install -r req.txt   │
    │                               └── load_plugin_from_path()     │
    │                                   └── load_plugin() ──────────┤
    │                                       ├── importlib 动态导入   │
    │                                       ├── 创建 PluginApi      │
    │                                       ├── register_manifest()  │
    │                                       └── plugin.register(api) │
    │                                           ├── register_tool()  │
    │                                           ├── register_provider│
    │                                           └── register_startup │
    │                                                               │
    ├── app 启动 ──────────────────┤                               │
    │                    PluginLoader.load_all_plugins()             │
    │                        └── 逐个 load_plugin()                  │
    │                                                               │
    │                    registry.set_runtime_helpers()              │
    │                    registry.register_providers()               │
    │                    注册控制命令                                 │
    │                    执行 startup hooks ────────────────────────┤
    │                        ├── (tool 注入 agents.tools)            │
    │                        ├── (cloudpaw 初始化)                   │
    │                        └── (pet 启动)                          │
    │                                                               │
    ├── 运行时 ─────────────────────────────────────────────────────┤
    │    LLM 调用 tool → agents.tools.xxx()                          │
    │    get_tool_config("xxx") → registry.get_tool_config()         │
    │    HTTP 请求 → /api/{prefix} → plugin router                   │
    │    /command → plugin handler                                    │
    │                                                               │
    ├── uninstall(plugin_id) ───────┤                               │
    │                    unload_plugin()                              │
    │                        ├── shutdown hooks                      │
    │                        ├── sys.modules.pop()                   │
    │                        ├── registry.unregister_plugin() ──────┤
    │                        │   ├── 移除 HTTP 路由                  │
    │                        │   ├── 移除 providers/hooks/commands   │
    │                        └── shutil.rmtree(delete_files)          │
```

---

## 五、pyproject.toml 入口点扩展

`pyproject.toml` 中预留了 setuptools entry-points 机制但**当前未启用**：
```toml
# [project.entry-points."qwenpaw.doctor"]
# my_pkg = "my_pkg.doctor:doctor_notes"
# Legacy group "copaw.doctor" is still loaded when present.
```

当前插件发现完全是**文件系统驱动**的：扫描 `plugins/<id>/plugin.json`。

---

## 六、关键设计决策

1. **延迟工具注册**：工具通过 startup hook 注入，确保应用初始化完成后再注册
2. **importlib 动态导入**：每个插件使用独立模块名 `plugin_{id}`，避免命名冲突；`submodule_search_locations` 支持相对导入
3. **pip → uv 回退**：依赖安装自适应不同 Python 环境
4. **HTTP 路由优先级**：插件路由插入到 SPA catch-all 之前，确保匹配优先
5. **配置按 Agent 隔离**：工具配置存储在每个 agent 的配置文件中，`get_current_agent_id()` 确定上下文
6. **启动/关闭钩子优先级**：使用整数优先级排序，低值先执行
7. **非侵入式**：插件的 `register()` 方法在加载时调用，所有初始化延迟到 startup hook
