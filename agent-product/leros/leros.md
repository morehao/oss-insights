# Leros 后端系统深度分析

## 1. 项目概览

Leros 是一个"AI OS"——企业级 AI 工作协作平台。核心理念：**给 AI 一个项目、一个角色、一个任务，它独立执行，你验收结果**。

- **语言**: Go 1.25
- **模块路径**: `github.com/insmtx/Leros`
- **HTTP 框架**: Gin
- **消息队列**: NATS JetStream
- **ORM**: GORM（支持 SQLite / PostgreSQL / MySQL / ClickHouse）
- **LLM 运行时**: CloudWeGo Eino（原生，`agent/runtime/native/`）+ Claude Code / Codex CLI / OpenCode（外部引擎，`agent/runtime/claude/` `agent/runtime/codex/` `agent/runtime/opencode/`）
- **消息契约**: `pkg/messaging/`（统一 Envelope + WorkerCommand + RunEvent 类型）
- **前端**: Next.js + Electron（`frontend/` 目录）

---

## 2. 启动方式

### 2.1 本地开发

```bash
make dev-setup   # 初始化开发环境
make dev-server  # 启动 Server（API + NATS 发布端）
make dev-worker  # 启动 Worker（任务消费 + Agent 运行时）
```

### 2.2 分布式生产

```bash
# Server：提供 HTTP API + 接收外部事件
leros server --config /path/to/config.yaml

# Worker：处理异步任务
leros worker --worker-id 1 \
  --config /path/to/config.yaml \
  --server-addr "127.0.0.1:8080" \
  --listen-addr ":8081" \
  --default-runtime native
```

### 2.3 调试工具

```bash
leros chat       # CLI 聊天调试
leros project    # 项目管理调试
leros session    # 会话管理调试
leros task       # 任务管理调试
```

---

## 3. 架构模型：四平面

Leros 采用**四平面架构**，将系统按职责垂直切分：

```
┌─────────────────────────────────────────┐
│              Edge Plane                  │  ← 客户端/边缘
│   App / CLI / 本地 Agent Runtime        │
└───────────────────┬─────────────────────┘
                     │ WebSocket / HTTP
                     ▼
┌─────────────────────────────────────────┐
│          Interface Layer                 │  ← 接口层
│   Assistant Service / Connector         │
└───────────────────┬─────────────────────┘
                     │ NATS
                     ▼
┌─────────────────────────────────────────┐
│          Control Plane                   │  ← 控制面
│   Event Engine / Memory / Policy Engine │
└───────────────────┬─────────────────────┘
                     │ NATS
                     ▼
┌─────────────────────────────────────────┐
│         Execution Plane                  │  ← 执行面
│   Agent Runtime / Skill / Tool          │
└─────────────────────────────────────────┘
```

- **Server** = Edge Plane + Interface Layer + Control Plane（入口，不执行）
- **Worker** = Execution Plane（只执行，不提供入口）

核心设计决策：**Server 和 Worker 通过 NATS 解耦**。Server 负责接收请求/编排，Worker 负责任务执行。这种分离带来了弹性伸缩——可以启动多个 Worker 挂载到同一个 NATS。

---

## 4. 核心接口

### 4.1 EventBus — 消息总线

**文件**: `backend/internal/infra/mq/bus.go`

```go
type EventBus interface {
    Publisher
    Subscriber
}
type Publisher interface {
    Publish(ctx, topic, event) error
}
type Subscriber interface {
    Subscribe(ctx, topic, consumer, handler) error
    SubscribeFrom(ctx, topic, startSeq, handler) error
}
```

### 4.2 agent.Runtime — Agent 运行时

**文件**: `backend/agent/runtime.go`

```go
type Runtime interface {
    Name() string
    Execute(ctx, req ExecutionRequest, observer Observer) (ExecutionResult, error)
}
```

四种实现：
- **native** — CloudWeGo Eino 框架（内置，始终可用），`agent/runtime/native/`
- **claude** — Claude Code CLI，`agent/runtime/claude/`
- **codex** — Codex CLI，`agent/runtime/codex/`
- **opencode** — OpenCode CLI，`agent/runtime/opencode/`

### 4.3 agent.Executor — 执行器

**文件**: `backend/agent/executor.go`

统一执行入口，负责 Registry 解析 Runtime + 生命周期管理（started/completed/failed/cancelled 事件）。

### 4.4 agent.Tool — 执行器工具

**文件**: `backend/agent/tool.go`

```go
type Tool interface {
    Definition() ToolDefinition
    Execute(ctx context.Context, input json.RawMessage) (string, error)
}
```

### 4.5 tools.Tool — 服务端工具

**文件**: `backend/tools/tool.go`

```go
type Tool interface {
    Name() string
    Description() string
    InputSchema() Schema
    Execute(ctx, input) (string, error)
}
```

### 4.6 messaging — 统一消息契约

**文件**: `backend/pkg/messaging/`

```go
// Envelope 是通用消息信封
type Envelope[T any] struct {
    ID        string
    Type      MessageType
    Trace     TraceContext
    Route     RouteContext
    Body      T
}

// WorkerCommand — Server → Worker 统一命令
type WorkerCommand = Envelope[WorkerCommandBody]

// RunEvent — Worker → Server/UI 统一运行事件
type RunEvent = Envelope[RunEventBody]
```

### 4.7 WorkerScheduler — Worker 调度器

**文件**: `backend/internal/worker/scheduler.go`

```go
type WorkerScheduler interface {
    Start(ctx, spec) (*WorkerInstance, error)
    Stop(ctx, workerID) error
    Health(ctx, workerID) error
    List(ctx) ([]*WorkerInstance, error)
}
```

三种实现：ProcessScheduler（子进程）、DockerCLIScheduler（Docker 容器）、KubernetesScheduler（K8s Pod）

---

## 5. 核心执行链路

### 5.1 完整业务流程

整个系统围绕一条核心链路运转：**消息输入 → 任务编排 → Worker 消费 → Agent 执行 → 流式响应 → 持久化完成**。

#### 5.1.1 消息输入阶段

```
POST /v1/NewMessage
  │
  ├─ 1. resolveOrCreateProject()  — 自动创建/查找 Project
  │      → 指定 ProjectID 则查找，否则以消息内容前 50 字符创建
  │      → 同时创建 ProjectMember（owner 角色）
  │
  ├─ 2. ensureProjectSession()    — 确保 Project 级别 Session
  │      → 类型为 SessionTypeProject，用于项目协作
  │
  ├─ 3. resolveOrCreateTask()     — 自动创建/查找 Task
  │      → 指定 TaskID 则查找，否则创建新 Task
  │      → 状态流转: created → pending → in_progress → completed
  │
  ├─ 4. createTaskSession()       — 创建 Task 级别 Session
  │      → 类型为 SessionTypeTask，绑定到 Task 和 Project
  │      → 双向绑定: session.task_id = task.id, task.session_id = session.id
  │
  ├─ 5. PostMessage()             — 持久化消息
  │      ├─ GetNextSequence()     — 获取递增序号
  │      ├─ CreateMessage()       — 写入 DB（role=user, status=pending）
  │      ├─ IncrementMessageCount() + UpdateLastMessageAt() — 更新统计
  │      ├─ Publish(SessionMessageRequestSubject) — 通知标题自动生成
  │      └─ publishWorkerTask()   — 构建 WorkerTaskMessage 发布到 NATS
  │           ├─ InferAssignedAssistantID() — 推断 Worker（默认 1）
  │           ├─ 构建 protocol.WorkerTaskMessage（含 Trace/Route/Body）
  │           ├─ 注入 LLM Model 配置（从 DB 读取组织默认模型）
  │           └─ Publish(org.{id}.worker.{id}.task, taskMsg)
  │
  └─ 返回 {project_id, task_id, session_id, message_id}
```

#### 5.1.2 Worker 消费阶段

```
NATS Message → handleEvent()
  │
  ├── dispatcher 按 lane 分发
  │    ├── cmd.run         → RunHandler.HandleRunCommand()
  │    │    ├─ 反序列化为 messaging.WorkerCommand
  │    │    ├─ 校验路由 (orgID/workerID 匹配)
  │    │    ├─ 校验 ModelConfig (provider, model, api_key 必填)
  │    │    ├─ SeqTracker (SQLite): TrackReceived → 去重判断
  │    │    ├─ RunCoordinator (Session 级别去抖 1.5s)
  │    │    │    ├─ 首条消息等待执行完成
  │    │    │    └─ 后续消息在窗口内合并
  │    │    └─ 提交 WorkerPool (maxConcurrency=20)
  │    │         └─ assistant.Service.Run()
  │    │              ├─ 1. Validate — 请求校验
  │    │              ├─ 2. Preparer — 构建 ExecutionRequest
  │    │              │    ├─ Workspace (工作目录)
  │    │              │    ├─ Skill Catalog (技能注册)
  │    │              │    ├─ Memory (上下文记忆)
  │    │              │    └─ Tool Registry (工具注册)
  │    │              ├─ 3. agent.Executor.Execute() — 委托 Runtime
  │    │              │    └─ Registry.Resolve(kind) → Runtime.Execute()
  │    │              └─ 4. Finalizer — 后处理
  │    │                   ├─ Artifact (产物归档)
  │    │                   ├─ Git Commit
  │    │                   └─ 业务终态通知
  │    │
  │    ├── cmd.control     → ControlHandler (取消运行)
  │    ├── cmd.interaction → InteractionHandler (审批/问答)
  │    └── cmd.skill       → SkillHandler (技能管理)
  │
  ├── 事件发布 (NATSEventSink)
  │    ├── run.stream lane — 高频增量 (message.delta, reasoning.delta, tool_call.*)
  │    └── run.state lane  — 低频关键状态 (run.started, artifact.declared, terminal)
  │
  └── Worker 端 MCP Server
       └── 为外部 CLI 引擎提供 MCP 协议端点
```

#### 5.1.3 Agent 执行阶段

```
assistant.Service.Run(ctx, req)
  │
  ├─ 1. Validate — 校验 RunRequest
  │
  ├─ 2. Preparer (interface) — 准备执行上下文
  │    ├─ Workspace: 创建工作目录 + 检出代码
  │    ├─ Skills: 加载技能定义 → skills 目录
  │    ├─ Memory: 加载上下文记忆
  │    ├─ Tools: 注册内置工具 (8个) + 技能工具
  │    └─ 构建 ExecutionRequest {Runtime, ExecutionID, Input, ToolDefs, ...}
  │
  ├─ 3. agent.Executor.Execute(ctx, request, observer)
  │    ├─ Registry.Resolve(kind) → 按 Runtime.Kind 分发
  │    │
  │    ├── "native" → agent/runtime/native/ (Eino)
  │    │    ├─ 创建 ChatModel (Provider/APIKey/Model/BaseURL)
  │    │    ├─ 构建 8 个 Tool Binding
  │    │    │    ├─ memory          — 记忆读写
  │    │    │    ├─ skill_use       — 技能使用
  │    │    │    ├─ skill_manage    — 技能管理
  │    │    │    ├─ todo            — TODO 管理
  │    │    │    ├─ node_shell      — Shell 执行
  │    │    │    ├─ node_file_read  — 文件读取
  │    │    │    ├─ node_file_write — 文件写入
  │    │    │    └─ artifact_declare — 产物声明
  │    │    ├─ Eino Flow (MaxStep=90)
  │    │    └─ StreamWithUsage → eventsCh → observer.Emit()
  │    │
  │    ├── "claude" → agent/runtime/claude/ (ExternalCLI)
  │    │    ├─ Engine.Prepare(workDir)
  │    │    ├─ Setup MCP server + tools
  │    │    ├─ Run(req) → Events channel
  │    │    └─ consumeEvents() → observer.Emit()
  │    │
  │    ├── "codex" → agent/runtime/codex/ (同上)
  │    │
  │    └── "opencode" → agent/runtime/opencode/ (同上)
  │
  └─ 4. Finalizer (interface) — 后处理
       ├─ ArtifactReconcile — 产物核对
       ├─ Git Commit — 自动提交代码变更
       └─ 业务终态通知
```

#### 5.1.4 流式响应阶段

```
NATSEventSink.Emit(event)
  │
  ├─ 映射 agent.Event → messaging.RunEvent
  │    ├─ event type → RunEventType (run.started, message.delta 等)
  │    ├─ payload 结构化 (ToolCallPayload, TodoItem 等)
  │    └─ 终端事件附加 RunCompletedPayload
  │
  ├─ ClassifyRunEvent → lane
  │    ├── run.stream — 高频增量
  │    │    message.delta, reasoning.delta, message.completed,
  │    │    tool_call.started, tool_call.finished,
  │    │    todo.snapshot, todo.updated
  │    │    → org.{id}.session.{id}.run.stream
  │    │
  │    └── run.state — 低频关键状态
  │         run.started, run.completed, run.failed, run.cancelled,
  │         artifact.declared, approval.*, question.asked/answered
  │         → org.{id}.session.{id}.run.state
  │
  └─ 发布到 JetStream SESSION_RUN_STREAM
       │
       ├─ Server 端 Runnable 订阅:
       │    ├─ session-run-state-projector
       │    │    └─ 消费 run.state: 持久化运行状态 + 消息完成
       │    └─ run.stream projector
       │         └─ 消费 run.stream: 持久化产物声明
       │
       └─ Server 端 SSE Handler 订阅 run.stream + run.state
            ├─ StreamSessionEvents goroutine
            └─ 通过 SSE 推送到前端
```

支持的事件类型：
- `run.started` / `run.completed` / `run.failed` / `run.cancelled`
- `message.delta` — 消息流式片段
- `reasoning.delta` — 推理过程流
- `message.completed` — 单次消息完成
- `tool_call.started` / `tool_call.finished`
- `todo.snapshot` / `todo.updated`
- `artifact.declared`
- `approval.requested` / `approval.resolved`
- `question.asked` / `question.answered`
- `work.title.updated`

---

### 5.2 GitHub Webhook 事件链路

```bash
GitHub → POST /api/github/webhook
  │
  ├── github.Connector.handleWebhook()
  │    ├─ 签名验证 (HMAC-SHA256)
  │    ├─ github.ParseWebHook() 类型转换
  │    ├─ convertEvent() → pkg/event.Event
  │    └─ determineTopic() → interaction.github.{event}
  │
  └── publisher.Publish("interaction.github.issue_comment", event)
       │
       └── (注: 当前 GitHub 事件通过 NATS 发布,
              由 Event Gateway 或 Worker 端 Runnable 消费)
```

### 5.3 后台消费者 (Runnable)

系统在 Server 启动时启动 2 个后台消费者，订阅 SESSION_RUN_STREAM 的 run.state lane：

| Runnable | 订阅主题 | 用途 |
|---|---|---|
| session-run-state-projector | `org.*.session.*.run.state` | 监听 run.completed/failed/cancelled → 持久化回复；监听 artifact.declared → 持久化产物；监听 approval.*/question.* → 互动处理 |
| session-run-stream-projector | `org.*.session.*.run.stream` | 监听 run.stream 增量事件 → 持久化流式片段 |

---

## 6. 消息拓扑

### 6.1 NATS JetStream Stream

| Stream | Subjects | 存储 | 保留策略 |
|--------|----------|------|----------|
| WORKER_CMD_STREAM | `org.*.worker.*.cmd.>` | FileStorage | 72h, 200/子主题 |
| SESSION_RUN_STREAM | `org.*.session.*.run.>` | FileStorage | 24h, 10000/子主题 |

### 6.2 Subject 命名规范

| Subject 模式 | Lane | 方向 | 用途 |
|---|---|---|---|
| `org.{id}.worker.{id}.cmd.run` | cmd.run | Server → Worker | Agent 执行命令 |
| `org.{id}.worker.{id}.cmd.control` | cmd.control | Server → Worker | 取消/控制命令 |
| `org.{id}.worker.{id}.cmd.interaction` | cmd.interaction | Server → Worker | 审批/问答 |
| `org.{id}.worker.{id}.cmd.skill` | cmd.skill | Server → Worker | 技能管理 |
| `org.{id}.session.{id}.run.stream` | run.stream | Worker → Server | 高频增量事件流 |
| `org.{id}.session.{id}.run.state` | run.state | Worker → Server | 低频关键状态 |
| `interaction.github.issue_comment` | — | GitHub → 事件引擎 | Issue 评论 |
| `interaction.github.pull_request` | — | GitHub → 事件引擎 | PR 事件 |
| `interaction.github.push` | — | GitHub → 事件引擎 | Push 事件 |

### 6.3 Consumer 名称

| Consumer | 订阅主题 | 用途 |
|---|---|---|
| `worker-run-consumer` | `org.*.worker.*.cmd.run` | Worker 执行命令消费 |
| `worker-control-consumer` | `org.*.worker.*.cmd.control` | Worker 控制命令消费 |
| `worker-interaction-consumer` | `org.*.worker.*.cmd.interaction` | Worker 互动命令消费 |
| `worker-skill-consumer` | `org.*.worker.*.cmd.skill` | Worker 技能管理命令 |
| `session-run-state-projector` | `org.*.session.*.run.state` | Server 端运行状态投影 |

---

## 7. API 路由全景

所有路由注册在 `backend/internal/api/router.go`，前缀 `/v1`：

### 7.1 业务 API

| 路由 | 用途 |
|------|------|
| `/v1/ws/*` | WebSocket 连接（实时消息推送） |
| `/v1/auth/*` | 认证（GitHub OAuth / JWT） |
| `/v1/assistants/*` | 数字员工 CRUD |
| `/v1/llm/models/*` | LLM 模型配置管理 |
| `/v1/sessions/*` | 会话管理（CRUD + 状态机） |
| `/v1/projects/*` | 项目管理 |
| `/v1/projects/{id}/files/*` | 项目文件浏览 |
| `/v1/work/*` | 工作入口（NewMessage 核心） |
| `/v1/tasks/*` | 任务管理 |
| `/v1/artifacts/*` | 产物管理 |
| `/v1/files/*` | 文件上传/下载 |
| `/v1/orgs/*` | 组织管理 |
| `/v1/users/*` | 用户管理 |
| `/v1/skills/marketplace/*` | Skill 市场 |
| `/v1/swagger/*` | Swagger API 文档 |

### 7.2 Worker 管理 API（非 v1 前缀）

| 路由 | 功能 |
|---|---|
| `GET /ws/worker` | Worker WebSocket 注册（心跳） |
| `POST /ListWorkers` | 列出活跃 Worker |
| `POST /GetWorkerInfo` | Worker 详情 |
| `POST /ShutdownWorker` | 关闭指定 Worker |
| `POST /CreateWorker` | 创建新 Worker 进程 |

---

## 8. 启动流程

### 8.1 Server 进程

```bash
1. 加载配置 (config.LoadConfig)
2. 初始化数据库 (infradb.InitDB)
     ├── GORM AutoMigrate (所有 19 个模型)
     └── 初始化系统数据 (SystemInitData)
3. 初始化文件存储 (filestore.Init)
4. 初始化 NATS (mq.NewNATS)
     └── 自动创建 TASK_STREAM + SESSION_STREAM
5. 初始化 Worker 调度器 (ProcessScheduler)
6. 组装 API 路由 (api.SetupRouter)
     ├── 中间件: CORS → Caller/Auth → Logger → Recovery
     ├── 注册所有 Handler
     ├── 注册 Worker Manager 路由
     └── 启动 3 个后台 Runnable (goroutine)
7. 启动 Gin HTTP Server
8. 等待退出信号 (lifecycle.Std().WaitExit())
```

### 8.2 Worker 进程

```bash
1. 加载配置 (config.LoadConfig)
2. 初始化 NATS (mq.NewNATS)
      └── 自动创建 WORKER_CMD_STREAM + SESSION_RUN_STREAM
3. 初始化 DB (infradb.InitDB)
4. 初始化文件存储 (filestore.Init)
5. 创建 Runtime Registry
      ├── 注册 native (必有)
      ├── 注册 claude (可选, 探测 which claude)
      ├── 注册 codex (可选, 探测 which codex)
      └── 注册 opencode (可选, 探测 which opencode)
6. 创建 assistant.Service
      ├── Preparer (workspace/skill/memory/tool)
      ├── agent.Executor (带 Registry)
      └── Finalizer (artifact/git/业务终态)
7. 创建 WorkerClient
      └── WebSocket 连接 Server → 注册心跳
8. 创建 Command Dispatcher (NATS 4 lane 订阅)
      ├── cmd.run → RunHandler
      ├── cmd.control → ControlHandler
      ├── cmd.interaction → InteractionHandler
      └── cmd.skill → SkillHandler
9. 创建 RunCoordinator (去抖/并发/取消)
10. 创建 MCP Server (Worker 端)
11. 等待退出信号
```

Worker 的 Runtime Registry 注册流程（Bootstrap 内联在每个 Runtime 的初始化中）：

```
Runtime Registry 初始化
  │
  ├── 1. native Runtime (始终注册)
  │     └─ 基于 CloudWeGo Eino，Go 原生执行
  │
  ├── 2. CLI Runtime Discovery (外部 CLI)
  │     ├── 检查 claude CLI 是否存在（which claude）
  │     ├── 检查 codex CLI 是否存在（which codex）
  │     ├── 检查 opencode CLI 是否存在（which opencode）
  │     └── 为每个可用 CLI 创建 ExternalCLI Runtime
  │
  └── 3. Skill 同步
        ├── 读取 workspace skills 目录
        ├── 创建 symlink 到外部 CLI 的 skills 目录
        └── 写入 .leros/skills/ 索引
```

---

## 9. Worker 进程生命周期

```
Server 端                          Worker 端

POST /CreateWorker
   │
   ├─ ProcessScheduler.Start()
   │    └─ exec(leros worker ...)
   │
   │                         ┌────────────────────────────────────────────┐
   │                         │ WorkerClient.Start()                       │
   │                         │   ├─ 4 lane NATS Subscribe(cmd.*)          │
   │                         │   ├─ WebSocket → Server                    │
   │                         │   ├─ WorkerRegister 消息                    │
   │                         │   ├─ Start MCP Server                      │
   │                         │   └─ readLoop / writeLoop                  │
   │                         └────────────────────────────────────────────┘
   │                                              │
   │    WebSocket Ping (54s)                      │
   │    ←─────────────────────────────────────   │
   │                                              │
   │    Heartbeat Checker (30s, 超时90s)           │
   │                                              │
   │    ────────── 消息到来 ──────────────────→  │
   │                                              │
   │    Publish(org.{id}.worker.{id}.cmd.run)     │
   │    ────────────────────────────────────────→ │
   │                                         ┌──┴─────────────────────────┐
   │                                         │ Dispatcher.RunHandler      │
   │                                         │   decode → validate        │
   │                                         │   seqTracker → 去重        │
   │                                         │   RunCoordinator.Submit    │
   │                                         │   assistant.Service.Run()  │
   │                                         │   NATSEventSink.Emit()     │
   │                                         └────────────────────────────┘
   │                                              │
   │    流式事件 ←─── NATS (run.stream) ────────  │
   │    WebSocket → 前端                           │
   │                                              │
   │    ───────── ShutdownCommand ──────────────→  │
   │                                         ┌──┴─────────────────────────┐
   │    monitorProcess goroutine              │ Shutdown()                │
   │    ←──── 进程退出 ──────────────────    │   关闭 WebSocket + MCP   │
   │                                         │   关闭 NATS 订阅          │
   │                                         │   退出进程                 │
   │                                         └────────────────────────────┘
```

**WebSocket 心跳机制**：
- writePump：每 54s 发送 Ping
- readPump：ReadDeadline = 60s，PongHandler 刷新
- heartbeatChecker：每 30s 检查 LastSeen，超时 90s 断连
- Send 缓冲：256

---

## 10. 关键数字汇总

| 参数 | 默认值 | 位置 |
|---|---|---|---|
| Worker Pool 大小 | 20 | `internal/worker/run/coordinator.go` |
| RunCoordinator 全局并发上限 | 20 | `internal/worker/run/coordinator.go` |
| Debounce 窗口 | 1.5s | `internal/worker/run/coordinator.go` |
| NATS WORKER_CMD_STREAM 保留 | 72h, 200/子主题 | `pkg/messaging/subject.go` |
| NATS SESSION_RUN_STREAM 保留 | 24h, 10000/子主题 | `pkg/messaging/subject.go` |
| Eino Flow MaxStep | 90 | `agent/runtime/native/runner.go` |
| WebSocket Send 缓冲 | 256 | `internal/worker/server/conn.go` |
| WebSocket Ping 间隔 | 54s | `internal/worker/server/server.go` |
| WebSocket Read Deadline | 60s | `internal/worker/server/server.go` |
| WebSocket Heartbeat | 30s 检查, 90s 超时 | `internal/worker/server/server.go` |
| Runnable 重启延迟 | 5s | `internal/runnable/runner.go` |
| 会话标题自动生成 | 第三条消息前 | `internal/service/session_service.go` |
| 终端事件发布超时 | 5s | `internal/worker/eventpub/nats_sink.go` |

---

## 11. 项目结构

```
Leros/
├── backend/                   # Go 后端
│   ├── cmd/leros/             # 进程入口（Cobra CLI: server, worker, chat, session 等）
│   ├── config/                # 配置结构
│   ├── agent/                 # Agent 独立执行层（无业务依赖）
│   │   └── runtime/           # Agent 运行时实现
│   │       ├── native/        # Eino 原生 Runtime
│   │       ├── claude/        # Claude Code CLI
│   │       ├── codex/         # Codex CLI
│   │       ├── opencode/      # OpenCode CLI
│   │       ├── externalcli/   # 共享 CLI 进程设施
│   │       ├── provider/      # CLI provider 进程协议
│   │       └── events/        # 运行时事件类型
│   ├── types/                 # 领域模型（23+ 个 GORM 模型）
│   ├── internal/
│   │   ├── api/               # HTTP API（Gin 路由 + Handler + 中间件）
│   │   │   └── connectors/    # 渠道连接器（github/gitlab/wework）
│   │   ├── service/           # 业务服务（25+ 文件）
│   │   ├── assistant/         # 业务包装层（Preparer + Finalizer）
│   │   ├── worker/            # Worker 子系统（调度器/命令分发/协调器/协议）
│   │   │   ├── command/       # 四 lane 命令分发
│   │   │   ├── run/           # RunCoordinator（去抖/并发/取消）
│   │   │   ├── scheduler/     # process/docker/k8s 调度器
│   │   │   ├── server/        # Worker 服务端（HTTP + WS）
│   │   │   ├── client/        # Worker 客户端
│   │   │   ├── eventpub/      # agent.Event → NATS run.stream/run.state
│   │   │   ├── mcp/           # Worker MCP Server
│   │   │   ├── router/        # Worker HTTP 路由
│   │   │   └── wsproto/       # WebSocket 协议
│   │   ├── skill/             # 技能系统（catalog/manage/store）
│   │   ├── runnable/          # 后台消费者（state/stream 投影）
│   │   ├── workspace/         # 工作空间管理
│   │   ├── memory/local/      # 本地记忆存储
│   │   ├── modelrouter/       # LLM 模型代理（v2）
│   │   └── infra/             # 基础设施（DB/MQ/WebSocket/GitHub Provider）
│   ├── tools/                 # 工具系统（node/memory/skill/todo/artifact）
│   ├── pkg/                   # 共享包
│   │   ├── messaging/         # 统一消息契约（Envelope/WorkerCommand/RunEvent）
│   │   ├── event/             # 运行时事件结构
│   │   ├── eino/              # Eino ChatModel/Flow/Tool 适配
│   │   └── leros/             # 路径解析工具
│   ├── prompts/               # 提示词
│   ├── skills/                # 内置技能（server + worker）
│   └── tests/                 # 集成测试
├── frontend/                  # Next.js + Electron
├── docs/                      # 文档
└── deployments/               # Docker/部署
```

---

## 12. 主要依赖

| 依赖 | 用途 |
|------|------|
| `gin` | HTTP 框架 |
| `cobra` | CLI 框架 |
| `nats.go` | 消息队列 |
| `gorm` | ORM |
| `cloudwego/eino` | LLM 运行时框架（native） |
| `cloudwego/eino-ext` | Claude / OpenAI 模型接入 |
| `gorilla/websocket` | WebSocket |
| `mcp-go` / `mark3labs/mcp-go` | MCP 协议 |
| `prometheus/client_golang` | 监控 |
| `go.uber.org/zap` | 日志 |
| `swaggo/swag` | Swagger 文档 |
| `ergo/services/crypto` | 加解密 |
| `lmittmann/tint` | 彩色日志 |
| `olekukonko/tablewriter` | CLI 表格输出 |

---

## 13. 开发验证

```bash
# 构建
make build

# 测试
go test ./...

# 代码检查
go fmt ./... && go vet ./...

# 运行
make dev-server   # 启动 Server
make dev-worker   # 启动 Worker（新终端）

# 生成 Swagger 文档
make swagger
```

---

## 14. 快速上手指南

### 第一周：理解数据流

1. 启动项目：`make dev-setup && make dev-server`
2. 阅读入口：`cmd/leros/server.go`
3. 阅读路由：`internal/api/router.go`
4. 阅读消息流：`internal/service/message_poster.go`
5. 阅读 Worker：`cmd/leros/worker.go`

### 第二周：深入业务

1. Service 层：`internal/service/`
2. Handler 层：`internal/api/handler/`
3. Agent 执行：`agent/executor.go`
4. Runtime 引擎：`agent/runtime/`

---

## 15. 核心代码索引

| 想了解什么 | 看哪个文件 |
|---|---|
| 项目启动入口 | `cmd/leros/main.go` |
| Server 初始化流程 | `cmd/leros/server.go` |
| Worker 初始化流程 | `cmd/leros/worker.go` |
| 配置结构定义 | `config/config.go` |
| NATS 消息总线 | `internal/infra/mq/bus.go` |
| 数据库初始化 + DAO | `internal/infra/db/` |
| API 路由表 | `internal/api/router.go` |
| 业务 Service | `internal/service/` |
| 消息投递编排 | `internal/service/message_poster.go` |
| Agent 执行入口 | `agent/executor.go` |
| Runtime 接口 | `agent/runtime.go` |
| Native Eino 引擎 | `agent/runtime/native/` |
| Claude 引擎 | `agent/runtime/claude/` |
| Codex 引擎 | `agent/runtime/codex/` |
| OpenCode 引擎 | `agent/runtime/opencode/` |
| 业务运行包装 | `internal/assistant/service.go` |
| 工作空间准备 | `internal/assistant/preparer_impl.go` |
| 运行终结器 | `internal/assistant/finalizer_impl.go` |
| Worker 命令分发 | `internal/worker/command/dispatcher.go` |
| Worker 运行协调 | `internal/worker/run/coordinator.go` |
| Worker 事件发布 | `internal/worker/eventpub/nats_sink.go` |
| Worker 进程调度 | `internal/worker/scheduler/` |
| Worker 客户端 | `internal/worker/client/` |
| Worker 服务端 | `internal/worker/server/` |
| 统一消息契约 | `pkg/messaging/` |
| NATS Subject 定义 | `pkg/messaging/subject.go` |
| 消息信封 | `pkg/messaging/envelope.go` |
| 运行事件类型 | `pkg/messaging/event.go` |
| Worker 命令类型 | `pkg/messaging/command.go` |
| Tool 接口 | `tools/tool.go` |
| 领域模型 | `types/` |
| 常量枚举 | `types/constants.go` |
| 后台消费者 | `internal/runnable/` |
| DB DAO 层 | `internal/infra/db/` |
| GitHub Connector | `internal/api/connectors/github/` |
| 技能系统 | `internal/skill/` |
| 提示词系统 | `prompts/` |
| 内置技能 | `skills/` |
