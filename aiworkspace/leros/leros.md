# Leros 后端系统深度分析

## 1. 项目概览

Leros 是一个"AI OS"——企业级 AI 工作协作平台。核心理念：**给 AI 一个项目、一个角色、一个任务，它独立执行，你验收结果**。

- **语言**: Go 1.25
- **模块路径**: `github.com/insmtx/Leros`
- **HTTP 框架**: Gin
- **消息队列**: NATS JetStream
- **ORM**: GORM（支持 SQLite / PostgreSQL / MySQL / ClickHouse）
- **LLM 运行时**: CloudWeGo Eino（原生）+ Claude Code / Codex CLI（外部引擎）
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

### 4.2 Runner — Agent 运行器

**文件**: `backend/internal/agent/runner.go`

```go
type Runner interface {
    Run(ctx, req *RequestContext) (*RunResult, error)
}
```

Agent 运行的统一入口。无论触发源是 HTTP API、GitHub Webhook 还是 NATS 消息，最终都通过这个接口下发。

### 4.3 Engine — 外部 AI CLI 引擎

**文件**: `backend/engines/engine.go`

```go
type Engine interface {
    Prepare(ctx, req PrepareRequest) error
    RegisterMCP(ctx, cfg MCPServerConfig) error
    GetSkillDir() string
    Run(ctx, req RunRequest) (*RunHandle, error)
}
```

支持三种引擎：
- **native** — CloudWeGo Eino 框架（内置，始终可用）
- **claude** — Claude Code CLI
- **codex** — Codex CLI

### 4.4 Tool — 工具

**文件**: `backend/tools/tool.go`

```go
type Tool interface {
    Name() string
    Description() string
    InputSchema() Schema
    Execute(ctx, input) (string, error)
}
```

### 4.5 WorkerScheduler — Worker 调度器

```go
type WorkerScheduler interface {
    Start(ctx, spec) (*WorkerInstance, error)
    Stop(ctx, workerID) error
    Health(ctx, workerID) error
    List(ctx) ([]*WorkerInstance, error)
}
```

两种实现：ProcessScheduler（子进程）和 DockerCLIScheduler（Docker 容器）

---

## 5. 核心执行链路

### 5.1 完整业务流程

整个系统围绕一条核心链路运转：**消息输入 → 任务编排 → Worker 消费 → Agent 执行 → 流式响应 → 持久化完成**。

#### 5.1.1 消息输入阶段

```
POST /v1/work/new-message
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
  ├── [第一层] Semaphore (chan struct{}, 容量=40 = MaxConcurrency×2)
  │      └─ 阻塞式背压：信号量满时 NATS 消息不 ACK，阻止后续投递
  │
  ├── decodeWorkerTask() → 反序列化为 WorkerTaskMessage
  ├── validateRoute() → 校验 orgID/workerID 匹配
  ├── 校验 taskType == TaskTypeAgentRun
  ├── 校验 ModelConfig（provider, model, api_key 必填）
  │
  ├── SeqTracker (SQLite): TrackReceived → 去重判断
  │      └─ 存储 seq 到 Metadata._seqs
  │
  ├── [第二层] Trailing Debouncer (1.5s 窗口)
  │      └─ 按 sessionKey(orgID:workerID:sessionID) 合并
  │           - 首条消息等待执行完成（创建 waiter channel）
  │           - 后续消息在窗口内合并后立即返回
  │           - 合并策略：合并 Messages + Attachments，累积 _seqs
  │
  └── [第三层] WorkerPool (maxConcurrency=20)
         └─ executeWithTracker → runTask()
              ├─ RequestFromWorkerTask() → agent.RequestContext
              ├─ prepareWorkspace() → 工作目录初始化
              ├─ req.EventSink = NewMQStreamSink(...)
              └─ runner.Run(ctx, req)
```

#### 5.1.3 Agent 执行阶段

```
runner.Run(ctx, req)
  │
  └── RuntimeService.Run()
       │
       └── Lifecycle Runner (13步管道)
            │
            ├─ 1. NormalizeStep      — 请求归一化
            ├─ 2. JournalStep        — 日志初始化
            ├─ 3. ModelStep          — 模型配置注入
            ├─ 4. ContextStep        — 上下文构建（history + workspace）
            ├─ 5. AuthorizeStep      — 授权检查
            ├─ 6. StartEventStep     — 发送启动事件
            ├─ 7. ArtifactBaselineStep — 产物基线快照
            ├─ 8. ExecuteStep        — 委托 RuntimeRouter 执行
            ├─ 9. ArtifactReconcileStep — 产物协调
            ├─10. ArtifactStep        — 产物记录
            ├─11. PersistStep         — 持久化
            ├─12. LearningStep        — 学习记录
            └─13. SessionCompleteStep — 完成通知
```

ExecuteStep 内部通过 RuntimeRouter 按 `Runtime.Kind` 分发到不同引擎：

```
RuntimeRouter
  │
  ├── "leros"  → Native Engine (Eino)
  │    ├─ 创建 ChatModel (Provider/APIKey/Model/BaseURL)
  │    ├─ 构建 SystemPrompt + 8个 Tool Binding
  │    │    ├─ memory          — 记忆读写
  │    │    ├─ skill_use       — 技能使用
  │    │    ├─ skill_manage    — 技能管理
  │    │    ├─ todo            — TODO 管理
  │    │    ├─ node_shell      — Shell 执行
  │    │    ├─ node_file_read  — 文件读取
  │    │    ├─ node_file_write — 文件写入
  │    │    └─ artifact_declare — 产物声明
  │    ├─ Eino Flow (MaxStep=90)
  │    └─ StreamWithUsage → eventsCh → eventSink.Emit()
  │
  ├── "claude" → ExternalCLI Runner
  │    ├─ Engine.Prepare(workDir)
  │    ├─ Engine.Run(req) → RunHandle{Events chan(256)}
  │    └─ consumeEvents() → eventSink.Emit()
  │
  └── "codex"  → ExternalCLI Runner (同上)
```

#### 5.1.4 流式响应阶段

```
MQStreamSink.Emit(event)
  │
  ├─ 组包: MessageStreamMessage + StreamBody{Seq, Event, Payload}
  │
  ├─ Publish(SessionResultStreamSubject(orgID, sessionID))
  │    → org.{id}.session.{id}.message.stream
  │    → Server 端 SessionService.StreamSessionEvents 订阅
  │    → 通过 WebSocket 推送到前端
  │
  └─ 完成/失败时:
       Publish(SessionMessageCompletedSubject(orgID, sessionID))
       → org.{id}.session.{id}.message.completed
       → runnable.StartSessionCompleted (后台消费者)
            ├─ CompleteSessionMessage() — 持久化 assistant 回复
            ├─ 更新 message.chunks / artifacts / usage
            └─ 产物投影（清理敏感字段）
```

支持的事件类型：
- `run.started` / `run.completed` / `run.failed`
- `message.delta` — 消息流式片段
- `reasoning.delta` — 推理过程流
- `tool_call.started` / `tool_call.completed` / `tool_call.failed`
- `todo.snapshot` / `todo.updated`
- `artifact.declared`
- `approval.requested` / `approval.resolved`

---

### 5.2 GitHub Webhook 事件链路

```bash
GitHub → POST /api/github/webhook
  │
  ├── github.Connector.handleWebhook()
  │    ├─ 签名验证 (HMAC-SHA256)
  │    ├─ github.ParseWebHook() 类型转换
  │    ├─ convertEvent() → interactionevent.Event
  │    └─ determineTopic() → interaction.github.{event}
  │
  ├── publisher.Publish("interaction.github.issue_comment", event)
  │
  ├── eventengine.Orchestrator (NATS Subscribe goroutine)
  │    ├─ 反序列化 → interactionevent.Event
  │    ├─ handler: handleIssueComment/handlePullRequest/handlePush
  │    └─ runEvent(ctx, event)
  │         └─ requestFromInteractionEvent(event)
  │              └─ 构建 prompt (事件信封 + 任务指令 + context + payload)
  │                   runner.Run(ctx, requestContext)
  │
  └── agent.runtimeRouter.Run()
```

### 5.3 后台消费者 (Runnable)

系统在 Server 启动时启动 3 个后台 goroutine，使用 5s 延迟重启的消费循环：

| Runnable | 订阅主题 | 用途 |
|---|---|---|
| StartSessionArtifactDeclared | `org.*.session.*.message.stream` | 监听 artifact.declared 事件 → 持久化产物 |
| StartSessionCompleted | `org.*.session.*.message.completed` | 监听 run.completed/failed → 持久化回复 |
| StartSessionTitleHandler | `org.*.session.*.message.request` | 消息数<3 且未手动设置时 AI 自动生成标题 |

---

## 6. 消息拓扑

### 6.1 NATS JetStream Stream

| Stream | Subjects | 存储 | 保留策略 |
|--------|----------|------|----------|
| TASK_STREAM | `org.*.worker.*.task` + `org.*.worker.*.approval` + `org.*.worker.*.skill.install` | FileStorage | 72h, 200/子主题 |
| SESSION_STREAM | `org.*.session.*.message.*` | FileStorage | 24h, 10000/子主题 |

### 6.2 Subject 命名规范

| Subject 模式 | 方向 | 用途 |
|---|---|---|
| `org.{id}.worker.{id}.task` | Server → Worker | 任务投递 |
| `org.{id}.worker.{id}.approval` | Worker → Server | 审批请求 |
| `org.{id}.worker.{id}.skill.install` | Server → Worker | Skill 安装 |
| `org.{id}.session.{id}.message.request` | Server → 后台 | 消息请求通知 |
| `org.{id}.session.{id}.message.stream` | Worker → Server | 流式响应 |
| `org.{id}.session.{id}.message.completed` | Worker → Server | 完成通知 |
| `interaction.github.issue_comment` | GitHub → 事件引擎 | Issue 评论 |
| `interaction.github.pull_request` | GitHub → 事件引擎 | PR 事件 |
| `interaction.github.push` | GitHub → 事件引擎 | Push 事件 |

### 6.3 Consumer 名称

| Consumer | 订阅主题 | 用途 |
|---|---|---|
| `worker-task-consumer` | `org.*.worker.*.task` | Worker 任务消费 |
| `session-title-handler` | `org.*.session.*.message.request` | 会话标题自动生成 |
| `session-completed-handler` | `org.*.session.*.message.completed` | 会话完成处理 |
| `session-artifact-declared-handler` | `org.*.session.*.message.stream` | 产物声明持久化 |

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
3. 初始化 DB (infradb.InitDB)
4. 初始化文件存储 (filestore.Init)
5. Bootstrap (builtin.BootstrapService)
     ├── CLI Discovery — 探测 claude/codex CLI
     ├── Skill Sync — 同步 workspace skills 到 CLI 目录
     └── MCP Registration — 注册 MCP 服务器
6. 创建 Runtime Service
     └── 构建引擎注册表 (native 必有, 可选 claude/codex)
     └── 创建 Lifecycle Runner (13步管道)
7. 创建 WorkerClient
     └── WebSocket 连接 Server → 注册心跳
8. 创建 TaskConsumer (NATS 订阅)
     └── Semaphore(40) + Debouncer(1.5s) + Pool(20)
9. 等待退出信号
```

Bootstrap 的具体流程：

```
BootstrapService.Start()
  │
  ├── 1. CLIDiscoveryService
  │     ├── 检查 claude CLI 是否存在（which claude）
  │     ├── 检查 codex CLI 是否存在（which codex）
  │     └── 返回可用 CLI 列表
  │
  ├── 2. SkillSyncService
  │     ├── 读取 workspace skills 目录
  │     ├── 为每个可用 CLI 创建 symlink 到其 skills 目录
  │     └── 写入 .leros/skills/ 索引
  │
  └── 3. MCPRegistrarService
        ├── 为每个可用 CLI 启动 MCP 服务器
        └── 注册 MCP tool 列表
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
   │                         ┌──────────────────────────────┐
   │                         │ WorkerClient.Start()          │
   │                         │   ├─ NATS Subscribe(task topic)│
   │                         │   ├─ WebSocket → Server       │
   │                         │   ├─ WorkerRegister 消息       │
   │                         │   └─ readLoop / writeLoop     │
   │                         └──────────────────────────────┘
   │                                              │
   │    WebSocket Ping (54s)                      │
   │    ←─────────────────────────────────────   │
   │                                              │
   │    Heartbeat Checker (30s, 超时90s)           │
   │                                              │
   │    ─────────── 消息到来 ─────────────────→  │
   │                                              │
   │    Publish(org.{id}.worker.{id}.task)        │
   │    ────────────────────────────────────────→ │
   │                                         ┌──┴─────────────┐
   │                                         │ Task Consumer   │
   │                                         │ handleEvent()   │
   │                                         │ Runner.Run()    │
   │                                         │ MQStreamSink    │
   │                                         └────────────────┘
   │                                              │
   │    流式事件 ←─── NATS ─────────────────────  │
   │    WebSocket → 前端                           │
   │                                              │
   │    ───────── ShutdownCommand ──────────────→  │
   │                                         ┌──┴─────────────┐
   │    monitorProcess goroutine              │ Shutdown()     │
   │    ←──── 进程退出 ──────────────────    │  关闭 WebSocket│
   │                                         │  退出进程       │
   │                                         └────────────────┘
```

**WebSocket 心跳机制**：
- writePump：每 54s 发送 Ping
- readPump：ReadDeadline = 60s，PongHandler 刷新
- heartbeatChecker：每 30s 检查 LastSeen，超时 90s 断连
- Send 缓冲：256

---

## 10. 关键数字汇总

| 参数 | 默认值 | 位置 |
|---|---|---|
| Worker Pool 大小 | 20 | `internal/worker/taskconsumer/consumer.go` |
| Semaphore 容量 | 40 (= pool × 2) | `internal/worker/taskconsumer/consumer.go` |
| Debounce 窗口 | 1.5s | `internal/worker/taskconsumer/consumer.go` |
| NATS TASK_STREAM 保留 | 72h, 200/子主题 | `pkg/dm/stream.go` |
| NATS SESSION_STREAM 保留 | 24h, 10000/子主题 | `pkg/dm/stream.go` |
| Eino Flow MaxStep | 90 | `engines/native/runner.go` |
| Event Channel 缓冲 | 256 | `engines/native/runner.go` |
| WebSocket Send 缓冲 | 256 | `internal/worker/server/conn.go` |
| WebSocket Ping 间隔 | 54s | `internal/worker/server/server.go` |
| WebSocket Read Deadline | 60s | `internal/worker/server/server.go` |
| WebSocket Heartbeat | 30s 检查, 90s 超时 | `internal/worker/server/server.go` |
| Runnable 重启延迟 | 5s | `internal/runnable/runner.go` |
| 会话标题自动生成 | 第三条消息前 | `internal/service/session_service.go` |
| Event Context 截断 | 6000 字符 | `internal/eventengine/mapper.go` |

---

## 11. 项目结构

```
Leros/
├── backend/                   # Go 后端
│   ├── cmd/leros/             # 进程入口（Cobra CLI）
│   ├── config/                # 配置结构
│   ├── types/                 # 领域模型（19 个 GORM 模型）
│   ├── internal/
│   │   ├── api/               # HTTP API（Gin）
│   │   ├── service/           # 业务逻辑（25+ 文件）
│   │   ├── agent/             # Agent 运行接口
│   │   ├── runtime/           # Agent 运行时（Lifecycle Pipeline）
│   │   ├── worker/            # Worker 子系统
│   │   ├── eventengine/       # 事件引擎
│   │   ├── skill/             # 技能系统
│   │   ├── runnable/          # 后台消费者
│   │   ├── workspace/         # 工作空间
│   │   └── infra/             # 基础设施（DB/MQ/FileStore/WebSocket）
│   ├── engines/               # AI CLI 引擎适配
│   ├── tools/                 # 工具系统
│   ├── pkg/                   # 共享包
│   ├── prompts/               # 提示词
│   ├── skills/                # 内置技能
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
| `cloudwego/eino` | LLM 运行时框架 |
| `cloudwego/eino-ext` | Claude / OpenAI 模型接入 |
| `gorilla/websocket` | WebSocket |
| `mcp-go` / `mark3labs/mcp-go` | MCP 协议 |
| `prometheus/client_golang` | 监控 |
| `go.uber.org/zap` | 日志 |
| `swaggo/swag` | Swagger 文档 |
| `ygpkg/yg-go` | 内部库（日志、雪花ID、加解密、中间件） |

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
4. 阅读事件流：`internal/eventengine/orchestrator.go`
5. 阅读 Worker：`cmd/leros/worker.go`

### 第二周：深入业务

1. Service 层：`internal/service/`
2. Handler 层：`internal/api/handler/`
3. Agent 运行时：`internal/runtime/service.go`
4. Worker 系统：`internal/worker/taskconsumer/`

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
| Agent 运行接口 | `internal/agent/runner.go` |
| 运行时 Lifecycle Pipeline | `internal/runtime/lifecycle/runner.go` |
| Worker 任务消费 | `internal/worker/taskconsumer/consumer.go` |
| Worker 协议定义 | `internal/worker/protocol/` |
| Worker 进程调度 | `internal/worker/scheduler/` |
| Worker 客户端 | `internal/worker/client/` |
| Worker 服务端 | `internal/worker/server/` |
| 事件引擎（GitHub） | `internal/eventengine/orchestrator.go` |
| Engine 接口 | `engines/engine.go` |
| Native Eino 引擎 | `engines/native/` |
| Bootstrap 启动 | `engines/builtin/bootstrap.go` |
| Tool 接口 | `tools/tool.go` |
| NATS Stream 定义 | `pkg/dm/stream.go` |
| NATS Subject 模式 | `pkg/dm/subject.go` |
| 领域模型 | `types/` |
| 常量枚举 | `types/constants.go` |
| 后台消费者 | `internal/runnable/` |
| DB DAO 层 | `internal/infra/db/` |
