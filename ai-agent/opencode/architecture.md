# OpenCode 系统架构深读

> **关联文档:** [主解读](./README.md) · [LLM 四轴路由系统](./llm-route-system.md) · [Session V2 持久化编排](./session-system.md)

---

## 1. 架构总览

OpenCode 采用**分层 + 服务化**架构，核心包间依赖关系为：

```
Schema (纯合约, 浏览器安全)
    │
    ├──→ Protocol (HttpApi 定义)
    │       ├──→ Server (HTTP 服务实现)
    │       └──→ Client (代码生成 SDK)
    │
    └──→ Core (领域模型 + Effect 服务 + 数据库)
            │
            └──→ OpenCode (主入口: CLI/TUI/Server)
                    │
                    └──→ LLM (路由抽象层)
```

```
packages/
├── schema/         ← 纯合约, 61 个模块
├── protocol/       ← HttpApi 定义, 18 个 API 组
├── server/         ← 共享 HTTP 服务工具
├── client/         ← 生成的 SDK 代码
├── core/           ← 领域模型 + 数据库 + Effect 层
├── opencode/       ← CLI 入口 + Session + Tool + Server
├── llm/            ← LLM 路由 + Provider 适配
├── tui/            ← SolidJS OpenTUI 终端界面
├── app/            ← SolidJS Web 界面
├── desktop/        ← Electron 桌面
├── plugin/         ← 插件系统 SDK
└── httpapi-codegen/ ← 代码生成器
```

---

## 2. CLI 入口与命令体系

**入口文件:** `packages/opencode/src/index.ts`

使用 yargs 构建，注册 22+ 命令，覆盖所有使用场景：

| 类别 | 命令 | 说明 |
|------|------|------|
| **核心交互** | `run` | 运行交互式会话（主入口） |
| | `tui-thread` | TUI 交互模式 |
| | `serve` | 启动 HTTP Server |
| | `web` | Web 服务 |
| **配置管理** | `providers` | 提供商配置 |
| | `models` | 模型列表 |
| | `agent` | Agent 管理 |
| | `plugin` | 插件管理 |
| | `config` | 配置管理 |
| **会话操作** | `session` | 会话 CRUD |
| | `export` / `import` | 会话导入导出 |
| **集成** | `github` | GitHub 集成 |
| | `mcp` | MCP 配置 |
| | `attach` | 附加到进程 |
| **系统** | `upgrade` / `uninstall` | 升级卸载 |
| | `db` | 数据库操作 |
| | `debug` | 调试 |
| | `stats` | 统计 |

全局选项: `--print-logs`, `--log-level`, `--pure`

---

## 3. 启动流程

以 `serve` 命令为例的完整启动链路：

```
index.ts (yargs)
    │
    ├── cmd/serve.ts
    │   └── server.listen(opts)
    │
    ├── listenEffect(opts)
    │   └── startWithPortFallback(opts)     ← 端口回退
    │       └── listenerLayer(opts, port)
    │           └── HttpRouter.serve(
    │               HttpApiApp.createRoutes(opts)
    │           )
    │               └── Layer.provideMerge(
    │                   AppNodeBuilder.build(...)
    │               )
    │
    └── 服务初始化
        ├── AppRuntime.bootstrap()
        │   ├── Config 加载
        │   ├── 数据库初始化 (SQLite)
        │   ├── Plugin 加载
        │   └── Project 解析
        ├── SessionStore 初始化
        ├── ToolRegistry 加载 (Builtins + Plugins)
        └── HttpApi 路由注册
            ├── Health / Location / Agent / Session
            ├── Message / Model / Provider / Permission
            ├── FileSystem / Command / Skill / Event
            └── Question / Reference / Credential / PTY
```

`run` 命令的交互式流程：

```
index.ts → cmd/run.ts
    │
    ├── Location 创建 / 解析 project
    ├── Config 加载 → Agent 选择 → Model 选择
    ├── ToolRegistry 初始化
    ├── SessionV2.create() → 创建持久化会话
    │
    └── 主循环 (Agent Loop):
        UserInput → SessionV2.prompt()
            → admit (持久化 inbox)
            → wake → SessionRunner.drain()
                → promote (提升为可见消息)
                → ContextEpoch.prepare()
                → Model.resolve()
                → LLM.stream(request)
                → Tool 执行 / 工具结果
                → 继续或等待
```

---

## 4. Effect 层组合系统

这是项目最核心的架构设计——在 Effect-TS 原生 Layer 之上封装的 **LayerNode** 系统。

### 4.1 LayerNode 概念

**文件:** `packages/core/src/effect/layer-node.ts`

`LayerNode` 是对 Effect `Layer` 的包装，在原生能力上增加了：

| 能力 | 说明 |
|------|------|
| **类型安全依赖树** | 编译时检查依赖是否全部满足 |
| **Tag 分组** | 将节点分为不同"部署组"（Global / Location） |
| **Hoist** | 将某些节点提升到外层（如 Global 服务跨 Location 共享） |
| **编译** | 将节点树编译为单个 Effect Layer |

```ts
type Node<A, E, T> = {
  kind: "layer" | "unbound" | "group"
  name: string
  service?: Context.Service.Any
  implementation?: Layer.Any
  dependencies: readonly AnyNode[]
  tag?: T
}
```

### 4.2 两阶段部署

**定义在** `packages/core/src/effect/app-node.ts`

```
tags:
  location: ["global"]   ← location 服务可依赖 global
  global: []             ← global 是最底层
```

**Global 节点** — 进程级单例:
- 数据库连接
- 文件系统基础操作
- 配置系统
- 插件管理器

**Location 节点** — 每目录实例化（37 个服务节点）:
- Session 服务
- ToolRegistry
- Config（项目级配置）
- AgentV2 / CommandV2
- PermissionV2
- FileSystem
- Integration / PluginV2
- ...

**`LocationServiceMap`** 是一个 `LayerMap<Location.Ref, LocationServices>`，为每个打开的目录懒创建服务实例：

```
Location 打开 → 懒创建服务实例
    → 60 分钟不活跃自动回收
    → Location 关闭 → 释放所有资源
```

### 4.3 构建流程

**文件:** `packages/core/src/effect/app-node-builder.ts`

```ts
build(root, replacements?)
  → 如果 root 有未绑定的 LocationServiceMap，自动构建
  → LayerNode.compile(root, replacements)
  → 输出单个 Effect Layer
```

---

## 5. Tool 工具系统

### 5.1 两层架构

**Core 层** (`packages/core/src/tool/`):

| 文件 | 职责 |
|------|------|
| `tool.ts` | `Tool.make()` 工厂，创建不透明工具值 |
| `registry.ts` | 核心注册表，支持 Location + Application 两层注册 |
| `builtins.ts` | 内建工具定义（Location-scoped） |
| `application-tools.ts` | 进程级应用工具定义 |

**OpenCode 层** (`packages/opencode/src/tool/`):

实现 16+ 内置工具：

| 工具 ID | 权限 | 说明 |
|---------|------|------|
| `read` | read | 读取文件 |
| `write` | write | 写入文件 |
| `edit` | edit | 编辑文件 |
| `apply_patch` | edit | 应用 diff patch |
| `glob` | glob | 文件模式匹配 |
| `grep` | grep | 内容搜索 |
| `shell` | shell | 执行 Shell 命令 |
| `task` | task | 子 Agent 任务 |
| `webfetch` | webfetch | HTTP 请求 |
| `websearch` | websearch | 网页搜索 |
| `question` | question | 用户提问 |
| `todowrite` | todowrite | 待办事项 |
| `skill` | skill | 技能系统 |
| `lsp` | lsp | LSP 交互 |
| `plan_enter` / `plan_exit` | plan | 计划模式 |

### 5.2 三层注册模型

```
Builtin 工具 (Core 层定义)
    ↓
Application 工具 (进程级, 如 Plugin 注册)
    ↓
Location 工具 (项目级, 从 opencode.json 加载)
```

注册表在 `packages/opencode/src/tool/registry.ts` 中按 `providerID`、`modelID` 做工具选择过滤：

```ts
const layer = Layer.effect(Service, Effect.gen(function* () {
  const builtins = initBuiltins()
  const custom = loadCustomTools(config)     // tool/*.ts
  const plugin = loadPluginTools(plugins)    // Plugin API
  registry.register(builtins)
  registry.register(custom)
  registry.register(plugin)
  tools()  // 按 provider/agent 过滤
}))
```

---

## 6. Agent 系统

**文件:** `packages/opencode/src/agent/agent.ts`

### 6.1 内置 Agent

| Agent | 模式 | 说明 |
|-------|------|------|
| `build` | primary (默认) | 全能代理，可提问，可进入计划模式 |
| `plan` | primary | 计划模式，禁止编辑工具 |
| `general` | subagent | 通用子代理，用于并行多步骤任务 |
| `explore` | subagent | 快速代码探索，仅 grep/glob/read/bash |
| `compaction` | primary (hidden) | 会话压缩 |
| `title` | primary (hidden) | 会话标题生成 |
| `summary` | primary (hidden) | 会话摘要 |

### 6.2 Agent Schema

```ts
const Info = Schema.Struct({
  name: Schema.String,
  mode: Schema.Literals(["subagent", "primary", "all"]),
  permission: PermissionV1.Ruleset,  // 控制可用的工具
  model: Schema.optional(ModelSchema),
  prompt: Schema.optional(Schema.String),
  topP: Schema.optional(Schema.Finite),
  temperature: Schema.optional(Schema.Finite),
  steps: Schema.optional(Schema.Finite),
  // ...
})
```

用户通过 `opencode.json` 的 `agents.*` 配置可以覆盖或新增 Agent，每个 Agent 有独立的 `Permission.Ruleset` 控制工具访问权限。

---

## 7. Server 体系

### 7.1 双重路由

Server 使用 Effect `HttpApiBuilder` + 传统 `HttpRouter` 混合模式：

```
HttpApiApp
├── HttpApi.make("server")          ← API 声明层
│   ├── SessionGroup / MessageGroup
│   ├── ModelGroup / ProviderGroup
│   ├── ToolGroup / CommandGroup
│   ├── FileSystem / Permission
│   └── ...
│   └── .middleware(Authorization)
│       .middleware(SchemaErrorMiddleware)
│
└── HttpRouter.serve(HttpApi)       ← 路由执行层
```

### 7.2 API 组 (18 个)

| API 组 | 端点 | 说明 |
|--------|------|------|
| Session | CRUD + prompt/resume/interrupt | 会话生命周期 |
| Message | 查询/创建/更新 | 消息操作 |
| Model | 列表/查询 | 模型信息 |
| Provider | 列表/配置 | 提供商管理 |
| Agent | 列表/配置 | Agent 管理 |
| Permission | 检查/授予 | 权限管理 |
| FileSystem | 读/写/列表 | 文件系统 |
| Command | 执行 | Shell 命令 |
| Event | SSE 流 | 事件推送 |
| Health | 健康检查 | 服务状态 |
| PTY | 终端 | 伪终端 |

---

## 8. 数据流全景

```
┌──────────────────────────────────────────────────────────────────┐
│                        用户入口                                    │
│  CLI (run)    TUI (tui-thread)    Web (serve)    Desktop (app)   │
└──────────────────────────┬───────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Session V2 编排引擎                            │
│                                                                  │
│  1. prompt(id, sessionID, prompt, delivery)                      │
│     ├─ resolvePrompt → 标准化文件/MIME                           │
│     ├─ SessionInput.admit → 发布 PromptAdmitted 事件             │
│     │  └─ Projector → 写入 SessionInputTable                    │
│     └─ execution.wake → 调度执行                                 │
│                                                                  │
│  2. SessionRunner.drain()                                        │
│     └─ while-loop:                                               │
│        ├─ promoteSteers / promoteNextQueued                      │
│        ├─ ContextEpoch.prepare()                                 │
│        ├─ Model.resolve(session) → LLM Model                    │
│        ├─ History.entriesForRunner → 投影历史消息                 │
│        ├─ toLLMMessages → LLM 格式消息                           │
│        ├─ llm.stream(request) → LLMEvent Stream                  │
│        ├─ 每个事件: 发布持久化 SessionEvent                      │
│        │  └─ Projector → SessionMessageTable                     │
│        └─ Tool.settle → 工具执行结果                             │
│                                                                  │
│  3. 工具执行                                                     │
│     ├─ ToolRegistry.lookup → 按权限/Provider/Agent 过滤          │
│     ├─ 工具执行 (read/write/grep/shell/...)                      │
│     └─ 结果回写 → 继续或结束 turn                                │
└──────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                    LLM 路由层                                     │
│                                                                  │
│  LLMClient.stream(request)                                       │
│    ├─ compile(request)                                           │
│    │  ├─ resolveRequestOptions → 合并 route/model/request 配置   │
│    │  ├─ route.body.from(request) → Provider 原生 body           │
│    │  └─ validateWith(body.schema) → Schema 校验                 │
│    │                                                             │
│    ├─ route.prepareTransport(body, request)                      │
│    │  ├─ 构建 HTTP Request (URL + Headers + Body)                │
│    │  └─ 配置 Framing (SSE / Binary)                             │
│    │                                                             │
│    └─ transport.frames(prepared, request, runtime)               │
│       ├─ HTTP POST → 字节流                                      │
│       ├─ Framing.frame → 帧流                                    │
│       ├─ decodeEvent(frame) → 类型化事件                          │
│       └─ stream.step(state, event) → LLMEvent[]                  │
└──────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                        存储层                                     │
│                                                                  │
│  SQLite (bun:sqlite / node:sqlite)                               │
│    ├─ Drizzle ORM Schema                                         │
│    │  ├─ SessionTable, SessionMessageTable                       │
│    │  ├─ SessionInputTable, SessionContextEpochTable             │
│    │  └─ ProjectTable, AccountTable, ...                         │
│    │                                                             │
│    ├─ EventV2.publish → events 表 (事件溯源)                     │
│    └─ Projectors → 投影到读模型表                                 │
│                                                                  │
│  File Storage (Local / S3 / 可扩展)                              │
└──────────────────────────────────────────────────────────────────┘
```

---

## 9. 关键架构模式总结

| 模式 | 实现位置 | 说明 |
|------|---------|------|
| **分层依赖反转** | 全局 | Schema → Protocol → Server 方向严格一致 |
| **组合优于继承** | `packages/llm/src/route/` | 四轴可组合的 LLM 路由 |
| **事件溯源 (Event Sourcing)** | `packages/core/src/session/` | 所有 Session 变更通过 EventV2.publish |
| **CQRS** | `packages/core/src/session/projector.ts` | Projector 从事件流写读模型 |
| **依赖注入** | `packages/core/src/effect/` | LayerNode 类型安全依赖组合 |
| **作用域服务** | `packages/core/src/effect/location-services.ts` | Location 粒度服务生命周期 |
| **工厂方法** | `packages/llm/src/route/client.ts` | Route.make() 构建路由 |
| **策略模式** | `packages/llm/src/route/auth.ts` | Auth.bearer/header/custom/none |
| **装饰器/中间件** | `packages/protocol/src/api.ts` | HttpApi .middleware() |
| **代码生成** | `packages/httpapi-codegen/` | HttpApi → SDK |

---

> **关联文档:** [主解读](./README.md) · [LLM 四轴路由系统](./llm-route-system.md) · [Session V2 持久化编排](./session-system.md)
