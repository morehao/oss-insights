# OpenBee — 24/7 数字员工管理平台深度分析

> **仓库:** https://github.com/theopenbee/openbee
> **公司:** 杭州万训科技有限公司
> **许可证:** MIT
> **npm 包:** `@theopenbee/cli`

## 第一部分：快速概览

### 这是什么

OpenBee 是一个自托管的 24/7 数字员工（Digital Workforce）管理平台。它的核心理念是：**把 AI 智能体当成数字员工来管理**——Worker 有名字、有头像、有部门、有状态（空闲/工作中/错误），像管人一样管 Agent。

与 Leros/Multica 的"Agent 协作平台"定位不同，OpenBee 的定位更接近**对话式 AI 员工管理后台**——你通过 Web 控制台或 IM 平台（飞书/钉钉/企业微信/Telegram）和 Bee（蜂后）对话，Bee 理解你的需求后，直接执行或把任务分派给 Worker（工蜂）。整个交互是对话驱动的，而非 Issue/Project 驱动的。

### 解决的痛点

- **AI 不是永久的**——每次在终端里打字，用完就没了，没有持续的"在线"概念
- **没有统一入口**——开发者要在不同终端窗口之间切换，没有统一的调度中心
- **无法协作**——一个 Agent 做了一半的活，其他开发者看不到进展、无法接手
- **没有任务调度**——无法说"每天晚上 8 点跑一次代码审查"，然后就不用管了
- **平台碎片化**——开发者在飞书里讨论、在终端里写代码、在 CI 上看结果——OpenBee 把 AI Worker 嵌入已有的 IM 工作流

### 适用场景

- 想在飞书/钉钉/微信里直接指挥 AI Worker 的团队
- 需要 Worker 按定时（cron）或倒计时执行任务的场景
- 希望有统一的 Web 仪表盘监控 AI Worker 运行状态的运维场景
- 对"对话即调度"模型有兴趣的 AI 代理平台开发者

### 核心能力清单

| 能力 | 说明 |
|------|------|
| **Bee 蜂后对话** | Bee 作为中央 AI 协调者，接收来自 IM 平台的消息，自主执行或分派给 Worker |
| **Worker 数字员工** | Worker 有名字、状态、引擎配置、约束规则、部门归属，具有组织身份 |
| **任务调度** | 支持立即执行、倒计时、cron 三种任务类型，Worker 级串行队列 |
| **部门组织** | 树形部门结构，Worker 可挂载到多部门，部门递归查询 Worker |
| **消息网关** | 去重（platform_msg_id）、去抖（session-level debounce）、命令检测、消息合并 |
| **多平台集成** | 飞书、钉钉、企微、微信、Telegram、Linear、本地 Web Chat |
| **Web 控制台** | 仪表盘、Worker 管理、Session/Execution 查看、环境变量管理、系统配置 |
| **多 AI 引擎** | Claude Code、Codex CLI、Pi——三个后端引擎可同时启用，Bee/Worker 独立配置 |
| **RPC 工具系统** | AI Worker 通过 HTTP RPC 调用 20+ 个工具（发消息、查状态、管理 Worker/任务/部门） |
| **约束系统** | 给 Worker 设置 Scope 级别的 Key-Value 约束，AI 可读写 |
| **环境变量加密** | 加密存储环境变量，运行时注入 Worker 进程 |
| **Token 统计** | 按模型收集 Token 用量，仪表盘展示趋势图 |

### 技术栈一览

| 层面 | 技术 |
|------|------|
| **后端语言** | Go 1.25 |
| **Web 框架** | Gin v1.12.0 |
| **数据库** | SQLite（modernc.org/sqlite，纯 Go 无 CGo） |
| **CLI 框架** | Cobra v1.10.2 |
| **AI 引擎适配** | Claude Code CLI / Codex CLI / Pi（插件式接口） |
| **IM 平台 SDK** | 飞书 SDK、钉钉 Stream SDK、Telegram Bot API、gorilla/websocket（企微） |
| **日志** | zap |
| **定时** | robfig/cron/v3 |
| **前端** | React 19 + TypeScript 5 + Vite 6 + Tailwind CSS 4 |
| **前端 UI** | @base-ui/react + shadcn + recharts + lucide-react |
| **前端数据** | @tanstack/react-query 5 + i18next |
| **分发包** | npm 包（`@theopenbee/cli`）、Homebrew Tap、Scoop、deb/rpm、GoReleaser |

---

## 第二部分：架构深读

### 架构总览

OpenBee 的架构围绕 **"一个 Bee 蜂后 + 多个 Worker 工蜂"** 的隐喻设计，整体是一个单体 Go 进程：

```
Message Platforms (Feishu/DingTalk/WeCom/Telegram/WeChat/Linear)
        │
        ▼
┌──────────────────────┐
│   Message Gateway    │  去重、去抖、命令检测
│   (msgingest)        │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   Message Store      │  SQLite — 消息持久化
│   (SQLite)           │
└──────────┬───────────┘
           │
   ┌───────┴───────┐
   │               │
   ▼               ▼
┌────────┐  ┌──────────────┐
│ Feeder │  │ Scheduler    │  任务调度器
│ (Bee)  │  │ (Task)       │
└───┬────┘  └──────┬───────┘
    │              │
    │       ┌──────▼───────┐
    │       │ Task         │  按 Worker 串行分发
    │       │ Dispatcher   │
    │       └──────┬───────┘
    │              │
    ▼              ▼
┌──────────────────────────┐
│    AI Engine Adapters    │  Claude / Codex / Pi
│    (EngineAdapter)       │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│   Worker Process (CLI)   │  子进程方式运行
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│   RPC Server (/rpc/bee)  │  Bee/Worker 调用的 HTTP 工具端点
│   20+ tools               │
└──────────────────────────┘
             │
             ▼
┌──────────────────────────┐
│   Platform Senders       │ 发送消息回 IM 平台
│   (Outbound)             │
└──────────────────────────┘
```

### 核心架构决策

1. **单体进程 vs Server/Worker 分离**: 与 Leros 完全相反，OpenBee 将所有组件（API、Feeder、Scheduler、RPC）放在**同一个 Go 进程**中。Worker 作为子进程（`exec.Command`）启动，通过 HTTP RPC 与主进程通信。一条命令 `openbee server` 就启动了整个系统。

2. **Bee 作为协调者**: Bee 是所有消息的第一接收者。它本身也是一个 AI 进程，运行在 Claude/Codex/Pi 引擎上。消息到达时，Bee 决定是自己处理还是通过 `@WorkerName` 语法分派给 Worker。

3. **SQLite 单机存储**: 使用 SQLite 而不是 PostgreSQL，适合单机部署场景。数据库文件存储在 `~/.openbee/` 目录下。46 个迁移以 Go 代码内联方式管理，无需外部迁移工具。

4. **RPC 作为 Agent 的后门**: AI Worker 进程通过 HTTP 调用 `/rpc/bee/call` 来使用各种工具。RPC 认证通过 JWT Token 实现，Worker 启动时由 Manager 注入 Token。

### 项目结构总览

```
openbee/
├── cmd/openbee/                  # 入口
│   └── internal/cli/             # Cobra 命令实现
│       ├── root.go               # 根命令
│       ├── configcmd/            # openbee config — 交互式配置向导
│       ├── ctlcmd/               # openbee ctl — 管理命令
│       ├── daemoncmd/            # openbee server — 后台服务
│       ├── servicecmd/           # openbee service — 系统服务管理
│       ├── backupcmd/            # openbee backup — 备份恢复
│       └── upgradecmd/           # openbee upgrade — 自升级
├── internal/
│   ├── ai/                       # AI 引擎抽象层
│   │   ├── contracts.go          # EngineAdapter 接口 + 核心类型
│   │   ├── registry.go           # 引擎工厂注册表
│   │   ├── claude/               # Claude Code CLI 适配器
│   │   ├── codex/                # Codex CLI 适配器
│   │   └── pi/                   # Pi CLI 适配器
│   ├── api/                      # HTTP API Handler
│   ├── app/                      # 应用组装启动（DI 容器）
│   ├── domain/
│   │   ├── bee/                  # Bee 进程（Feeder）
│   │   ├── command/              # IM 命令处理（/status /clear /list 等）
│   │   ├── enginecfg/            # 引擎配置存储
│   │   ├── env/                  # 环境变量管理
│   │   ├── msgingest/            # 消息摄入网关
│   │   ├── session/              # 会话清理服务
│   │   ├── task/                 # 任务调度 + 分发
│   │   └── worker/               # Worker 生命周期管理
│   ├── infra/
│   │   ├── auth/                 # JWT 认证
│   │   ├── backup/               # 备份恢复
│   │   ├── config/               # YAML 配置
│   │   ├── crypto/               # 加密
│   │   ├── i18n/                 # 国际化
│   │   ├── logger/               # 日志
│   │   ├── media/                # 媒体处理（ffmpeg）
│   │   ├── model/                # 数据模型结构体
│   │   ├── store/                # SQLite 数据访问层
│   │   └── utils/                # 通用工具
│   ├── platform/                 # 消息平台集成
│   │   ├── dingtalk/             # 钉钉
│   │   ├── feishu/               # 飞书
│   │   ├── linear/               # Linear
│   │   ├── local/                # 本地 Web Chat
│   │   ├── telegram/             # Telegram
│   │   ├── wecom/                # 企业微信
│   │   └── weixin/               # 微信
│   ├── routes/                   # HTTP 路由注册
│   ├── rpc/                      # RPC 服务（Agent 工具调用）
│   └── tokenstat/                # Token 用量统计同步
├── web/                          # React SPA 前端
│   ├── src/
│   │   ├── pages/                # 页面组件
│   │   ├── components/           # 通用组件
│   │   ├── hooks/                # 自定义 Hooks
│   │   ├── lib/                  # API 客户端、类型定义、认证
│   │   └── locales/              # i18n 翻译
│   └── embed.go                  # 前端 dist 嵌入 Go 二进制
├── npm/                          # npm 发布包
├── Makefile                      # 构建自动化
├── go.mod / go.sum
├── CLAUDE.md                     # AI 设计约束
├── DESIGN.md / DESIGN.json       # 设计系统
└── PRODUCT.md                    # 产品愿景
```

---

## 第三部分：核心执行链路

### 3.1 对话驱动执行链路

核心链路：**消息输入 → Bee 处理 → Worker 分派 → 执行 → 回复**

```
IM 平台（飞书/钉钉等）
  │
  │ 用户发送消息
  ▼
┌─────────────────────────────────────┐
│ Gateway.Dispatch(msg)                │
│   ├─ 1. 去重检测 (seen map)          │
│   │      → 按 platform_msg_id 去重    │
│   │      → 两级旋转 (10k cap)        │
│   ├─ 2. 命令检测 (IsCommand)          │
│   │      → /status /clear /stop 等    │
│   │      → 直接路由到 command handler │
│   ├─ 3. 去抖合并                      │
│   │      → 同一 session 的消息在窗口内 │
│   │      → 合并 content 后批量写入    │
│   │      → 前 n-1 条标记 merged       │
│   └─ 4. 写入 DB + 发送 IngestedMsg    │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│ Feeder.Run (轮询)                     │
│   ├─ 1. Poll 间隔: 1s (硬编码)        │
│   ├─ 2. CountReceived 检查队列容量    │
│   │      → >500 warn                  │
│   ├─ 3. ClaimBatch(available slots)   │
│   │      → 信号量控制并发=5            │
│   └─ 4. processBeeGroup(key, msgs)    │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────┐
│ processBeeGroup                           │
│   ├─ 1. tryDirectDispatch(@workerName)   │
│   │      → 检测 @WorkerName 前缀         │
│   │      → 创建 Task(worker_id, msg_id)  │
│   │      → 标记 bee_processed            │
│   │      → 返回（Bee 不处理）              │
│   ├─ 2. 获取 SessionContext              │
│   │      → 按 session_key + engine 查找  │
│   │      → resume = session_id != ""     │
│   ├─ 3. 合并消息 (merged content)        │
│   ├─ 4. 构建 Prompt                      │
│   │      → 非 resume: 添加 Bee 系统前缀  │
│   │      → 序列化 message_meta + content  │
│   ├─ 5. 创建 Execution (DB)              │
│   │      → trigger_input = prompt        │
│   │      → 获取 exec.ID → logPath        │
│   ├─ 6. runner.Run(bee, prompt)          │
│   │      → 启动 AI 引擎子进程             │
│   │      → 超时控制 (context.Timeout)    │
│   │      → waitBeeOutput(Output channel) │
│   ├─ 7. ExtractResult(logPath)           │
│   │      → 从日志文件提取最终回复         │
│   ├─ 8. Upsert SessionContext            │
│   └─ 9. MarkBeeProcessed                 │
└──────────────────────────────────────────┘
```

### 3.2 任务调度链路

Worker 分派的任务（以及用户手动创建的定时任务）通过 Scheduler + Dispatcher 执行：

```
┌────────────────────────────────────────┐
│ Scheduler.Run (轮询)                   │
│   ├─ 1. PeekDueScheduledTasks          │
│   │      → 计算 scheduled 任务的       │
│   │         next_run_at (cron 计算)     │
│   ├─ 2. ClaimDueTasks (原子)           │
│   │      → UPDATE ... WHERE条件          │
│   │      → 更新 next_run_at             │
│   └─ 3. DispatchTask → dispatchCh       │
└──────────────────┬─────────────────────┘
                   │
                   ▼
┌────────────────────────────────────────────┐
│ Dispatcher.Run                             │
│   ├─ 按 Worker 建立串行队列                │
│   │   → workerQueues map[string]chan       │
│   │   → 每个 Worker 同一时间只跑一个任务    │
│   ├─ 创建 Execution (DB)                   │
│   ├─ WorkerManager.Start(worker, task)     │
│   │   ├─ 解析引擎 (resolveEngine)          │
│   │   ├─ 准备 Worker 工作目录              │
│   │   ├─ 注入环境变量 (加密解密)            │
│   │   ├─ 生成 RPC Token (JWT, 含 scope)   │
│   │   ├─ resolveEngineArgs (合并全局+worker)│
│   │   ├─ runner.Run(worker, prompt)        │
│   │   └─ 管理 activeProcesses map          │
│   └─ 跟踪执行结果                          │
└────────────────────────────────────────────┘
```

### 3.3 RPC 工具调用链路

```
Worker 进程 (Claude Code)
  │
  │ 需要发消息/查 Worker/创建任务...
  ▼
┌──────────────────────────────────┐
│ POST /rpc/bee/call               │
│ Authorization: Bearer <JWT>      │
│ Body: {"name":"send_message",    │
│        "args": {...}}            │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│ beeCallTool (switch dispatch)    │
│   ├─ checkWorkerScope (权限检查) │
│   └─ 按照 name 路由到具体函数     │
│       ├── send_message           │
│       │   ├─ 查找 Message (DB)   │
│       │   ├─ 查找 Platform Sender│
│       │   └─ sender.Send(ctx,   │
│       │        OutboundMessage)  │
│       ├── list_workers           │
│       │   └─ WorkerStore +       │
│       │      Department filter   │
│       ├── create_task            │
│       │   ├─ 校验 type/指令/时间  │
│       │   └─ TaskStore.Create()  │
│       └── ... (20+ tools)        │
└──────────────────────────────────┘
```

---

## 第四部分：启动流程

### 主进程启动

```
main() → cli.Execute()
  │
  └─ serverCmd()
       │
       ├─ 1. 加载配置 (config.Load)
       │      └─ 读取 config.yaml → Config 结构体
       │
       ├─ 2. 初始化 SQLite (store.NewDB)
       │      └─ 运行 46 个迁移
       │
       ├─ 3. 初始化 AI Engine Registry
       │      ├─ 注册 claude (可选)
       │      ├─ 注册 codex (可选)
       │      └─ 注册 pi (可选)
       │
       ├─ 4. 初始化 Store 层 (所有 XXXStore)
       │
       ├─ 5. 初始化 Platform 集成
       │      ├─ Receiver (接收消息 → Gateway.Dispatch)
       │      └─ Sender (发送消息回平台)
       │
       ├─ 6. 初始化 Message Gateway
       │      └─ Gateway(msgStore, debounce, handler)
       │
       ├─ 7. 初始化 Feeder (Bee)
       │
       ├─ 8. 初始化 Task Scheduler
       │
       ├─ 9. 初始化 Task Dispatcher
       │
       ├─ 10. 初始化 Worker Manager
       │
       ├─ 11. 初始化 API Server (Gin)
       │       ├─ /api/* — RESTful API
       │       ├─ /rpc/bee/call — RPC 工具端点
       │       └─ JWT 认证中间件
       │
       ├─ 12. 启动后台 goroutine 集群
       │       ├─ Feeder.Run() — 消息消费
       │       ├─ Scheduler.Run() — 任务调度
       │       ├─ Dispatcher.Run() — 任务分发
       │       ├─ Gateway.Run() — 消息网关
       │       ├─ TokenStat Sync — Token 同步
       │       └─ 各 Platform Receiver
       │
       └─ 13. 启动 HTTP Server + 等待信号
```

---

## 第五部分：API 路由全景

### RESTful API（`/api/`）

| 路由 | 用途 |
|------|------|
| `POST /api/auth/login` | 登录（获取 JWT） |
| `POST /api/auth/refresh` | 刷新 Token |
| `GET /api/version` | 版本信息 |
| `GET/POST /api/workers` | Worker CRUD |
| `GET/PUT/DELETE /api/workers/:id` | 单个 Worker 操作 |
| `GET /api/workers/random-name` | 随机 Worker 名称生成 |
| `GET /api/sessions` | 执行记录列表 |
| `GET /api/sessions/:id` | Session 详情 |
| `GET /api/sessions/:id/logs` | 执行日志 |
| `GET /api/tasks` | 任务列表 |
| `DELETE /api/tasks/:id` | 取消任务 |
| `POST /api/workers/:id/tasks/cancel-all` | 取消 Worker 所有任务 |
| `POST/GET/PUT/DELETE /api/departments` | 部门 CRUD |
| `PUT /api/workers/:id/departments` | 设置 Worker 部门 |
| `GET /api/workers/:id/departments` | Worker 所属部门 |
| `GET /api/departments/:id/workers` | 部门下的 Worker |
| `POST/GET /api/local/messages` | 本地 Web Chat 消息 |
| `GET /api/local/stream` | 本地 Chat SSE 流 |
| `GET /api/messages` | 消息列表（过滤） |
| `GET /api/stats/overview` | 仪表盘统计数据 |
| `GET /api/stats/token-trend` | Token 趋势 |
| `GET/POST/PUT/DELETE /api/envs` | 环境变量管理 |
| `GET /api/system-configs` | 系统配置 |
| `PUT /api/system-configs/:key` | 更新系统配置 |

### RPC 工具端点（`/rpc/bee/call`）

| 工具名 | 用途 |
|--------|------|
| `list_workers` | 列出 Worker（支持部门过滤、递归） |
| `get_worker` | 获取单个 Worker 详情（含部门） |
| `create_worker` | 创建 Worker |
| `update_worker` | 更新 Worker |
| `delete_worker` | 删除 Worker |
| `create_task` | 创建任务（immediate/countdown/scheduled） |
| `list_tasks` | 列出任务（分页，含执行记录） |
| `cancel_task` | 取消任务 |
| `send_message` | 发送消息回 IM 平台 |
| `clear_session` | 清除 Bee 会话 |
| `get_worker_status` | 获取 Worker 运行状态 |
| `get_system_overview` | 系统概览统计 |
| `save/get/delete_constraint` | 约束 CRUD |
| `list_session_contexts` | 查看会话上下文 |
| `clear_worker_session` | 清除 Worker 会话 |
| `list_departments` | 部门列表（树形） |
| `get/create/update/delete_department` | 部门 CRUD |
| `list_messages` | 消息列表 |
| `list_outbound_messages` | 外发消息列表 |

---

## 第六部分：核心代码索引

| 想了解什么 | 看哪个文件 |
|---|---|
| 项目入口 | `cmd/openbee/main.go` |
| CLI 根命令 | `cmd/openbee/internal/cli/root.go` |
| 应用组装 | `internal/app/app.go` |
| 配置结构 | `internal/infra/config/config.go` |
| SQLite 迁移 | `internal/infra/store/db.go` |
| EngineAdapter 接口 | `internal/ai/contracts.go` |
| 引擎注册表 | `internal/ai/registry.go` |
| Claude 适配器 | `internal/ai/claude/engine.go` |
| Codex 适配器 | `internal/ai/codex/engine.go` |
| Pi 适配器 | `internal/ai/pi/engine.go` |
| 消息网关 | `internal/domain/msgingest/gateway.go` |
| Bee Feeder | `internal/domain/bee/feeder.go` |
| 任务调度器 | `internal/domain/task/scheduler.go` |
| 任务分发器 | `internal/domain/task/dispatcher.go` |
| Worker 管理器 | `internal/domain/worker/manager.go` |
| Worker 模型 | `internal/infra/model/worker.go` |
| 会话清理 | `internal/domain/session/clear_service.go` |
| API 路由 | `internal/routes/server.go` |
| API 定义 | `internal/routes/api.go` |
| RPC 路由 | `internal/routes/rpc.go` |
| RPC 工具实现 | `internal/rpc/tools.go` |
| RPC 服务端 | `internal/rpc/server.go` |
| JWT 认证 | `internal/infra/auth/auth.go` |
| Worker 数据访问 | `internal/infra/store/worker.go` |
| 任务数据访问 | `internal/infra/store/task.go` |
| 消息数据访问 | `internal/infra/store/message.go` |
| 消息平台接口 | `internal/platform/interfaces.go` |
| 飞书集成 | `internal/platform/feishu/` |
| 钉钉集成 | `internal/platform/dingtalk/` |
| Telegram 集成 | `internal/platform/telegram/` |
| 前端入口 | `web/src/main.tsx` |
| 前端路由 | `web/src/app.tsx` |
| 前端 API 客户端 | `web/src/lib/api.ts` |
| 前端类型定义 | `web/src/lib/types.ts` |

---

## 第七部分：同类项目对比

### 定位对比

| 维度 | OpenBee | Multica | Leros |
|------|---------|---------|-------|
| **定位** | 数字员工管理平台（对话驱动） | 托管式 Agent 协作平台（Issue 驱动） | 企业级 AI 工作平台（Session 驱动） |
| **核心隐喻** | 蜂后 + 工蜂（对话→分派） | Agent 作为 Issue 的 Assignee | 四平面架构 |
| **交付形态** | 单进程自托管 | 单进程 + 外部 Daemon | Server + Worker 双进程（NATS） |
| **Agent 类型** | Claude Code / Codex CLI / Pi | Claude Code / Codex CLI | Eino / Claude / Codex / OpenCode |
| **任务模型** | 消息→Bee→Worker + Task 调度 | Issue→AgentTaskQueue→Daemon | Session→Task→Worker（NATS） |
| **前端** | React SPA（Vite） | Next.js + Electron + RN | Next.js + Electron |
| **数据库** | SQLite（单机） | PostgreSQL + pgvector | SQLite / PG / MySQL / ClickHouse |
| **消息队列** | 无（单体进程） | 进程内事件总线 | NATS JetStream（必需） |
| **IM 集成** | 飞书/钉钉/企微/微信/Telegram | 飞书 | GitHub / 飞书 |
| **多租户** | ❌ 单用户 | ✅ Workspace 多租户 | ✅ Org 多租户 |

### 架构对比

| 维度 | OpenBee | Multica | Leros |
|------|---------|---------|-------|
| **Server-Worker 分离** | ❌ 单体 + 子进程 exec | ❌ 单体 + 外部 Daemon WS | ✅ 双进程 NATS 通信 |
| **Worker 通信** | HTTP RPC | WebSocket（Daemon Hub） | NATS（4 lane + 2 lane） |
| **分布式支持** | ❌ 单节点 | ⚠️ Redis 中继 | ✅ NATS 原生 |
| **内置 Agent 运行时** | ❌ 依赖外部 CLI | ❌ 依赖外部 CLI | ✅ 内置 Eino |
| **Workflow/Pipeline** | ❌ | ❌ | ✅ Eino Flow |
| **定时调度** | ✅ cron / countdown | ✅ Autopilot | ❌ |
| **实时推送** | ⚠️ SSE | ✅ 双 Hub WebSocket | ✅ NATS stream → SSE |
| **部门组织** | ✅ 树形部门 + 递归 | ✅ Squad 团队 | ❌ |
| **约束系统** | ✅ Scope-based KV | ❌ | ❌ |
| **Token 统计** | ✅ 按模型聚合 | ❌ | ❌ |
| **Skill 系统** | ❌ (AI 引擎自带) | ✅ Skill 市场 | ✅ Skill Catalog |
| **环境变量管理** | ✅ 加密 + 注入 | ❌ | ❌ |

### 关键差异

1. **单体 vs 分布式架构哲学**: OpenBee 坚定选择了单进程架构。一条命令启动所有服务，SQLite 数据库就在本地 `~/.openbee/` 目录下。没有任何外部依赖（不需要 NATS、不需要 PostgreSQL），特别适合单人/小团队自托管。Leros 把 Server 和 Worker 彻底拆成两个独立进程，依赖 NATS JetStream 通信，天然支持分布式多节点部署，运维复杂度也相应增加。

2. **对话即调度 vs Issue 即任务**: OpenBee 的核心交互模型来自 IM 平台消息——在飞书/钉钉里发消息给 Bee，Bee 理解后决定是自己做还是通过 `@WorkerName` 分派给 Worker。完全不需打开浏览器。Multica 以 Issue 为核心，用户在 Web 端创建 Issue → 分配 Agent → 执行 → 回传。Leros 以 Session 和 Project 为核心。

3. **AI Worker 的组织身份**: OpenBee 是唯一把"组织身份"做进 Worker 模型的平台。Worker 有名字、状态、描述、约束、工程目录、部门归属、权限范围。Bee 可以查询"研发部门有哪些闲置 Worker"然后把任务分给它们。这种设计赋予了 AI Worker 类似真实员工的身份和可见性。

4. **RPC 工具系统的完整性**: OpenBee 的 RPC 系统拥有 20+ 个工具，覆盖 Worker/部门/任务/消息/约束/会话的全生命周期管理。AI Worker 可以通过 RPC 反查数据库、创建新任务、甚至管理部门结构。这种"Agent 自管理"能力是其他项目没有的——在 Multica 和 Leros 中，Agent 是纯粹的"执行者"。

5. **平台集成深度**: OpenBee 集成了飞书/钉钉/企微/微信/Telegram/Linear 六大平台。消息去重、去抖、@提及剥离等 IM 特有的问题在 `msgingest` 网关中统一处理。Leros 主要依赖 GitHub Webhook，Multica 只集成飞书——OpenBee 在 IM 集成广度上远超两者。

---

## 第八部分：总结与评价

### 最值得学习的设计决策

1. **Gateway 消息处理流水线**: `msgingest.Gateway` 的去重 + 去抖 + 命令检测 + 消息合并流水线设计非常优雅。去重使用两级旋转 map（防止内存泄漏），去抖按 session 维度合并窗口内消息并标记前 n-1 条为 merged。单文件的 `gateway.go`（259 行）就完成了核心逻辑。

2. **Feeder 信号量并发控制**: `Feeder` 使用 Go channel 作为信号量控制并发 Bee 进程数（默认 5）。`tick()` 先检查可用槽位再 ClaimBatch，确保每个被 claim 的消息都能立即启动处理而不阻塞。

3. **tryDirectDispatch @Worker 分派**: 消息中检测 `@WorkerName` 前缀直接分派给 Worker，绕过 Bee 的 AI 推理过程，响应更快速且成本更低。

4. **会话恢复 (Session Resume)**: Bee 进程读取 `session_key + engine` 维度的 session_id。如果 session_id 存在，将 `Resume: true` 传递给引擎，实现跨消息的上下文连续性。

5. **Worker 串行执行队列**: `Dispatcher` 为每个 Worker 维护独立的串行 channel 队列，确保同一时间只执行一个任务。天然避免了并发冲突。

6. **Pre-flight Session Context 写入**: 在启动 AI 引擎前先写入 session context（即使 session_id 是新生成的），让 session_id 在进程启动前就对其他组件可见。

7. **RPC Worker Scope 权限控制**: RPC 工具基于 Worker Token 的 permission_scope 进行细粒度权限校验。Bee Token（无 worker_id）不受限制，Worker Token 只允许操作已授权的 scope。

### 潜在改进空间

- **缺乏分布式支持**: 单进程 + SQLite 架构限制了横向扩展能力。如果 Worker 数量增长到数十个，单机资源可能成为瓶颈。可以考虑引入可选的消息队列层（如 NATS）来解耦。
- **无内置 Agent 运行时**: 依赖外部 CLI（Claude Code / Codex CLI / Pi），意味着每个 Worker 都必须安装对应的命令行工具。缺乏 Leros 那样的内置 Eino 运行时。不过这也是设计取舍——OpenBee 的工作目录模式直接映射到 CLI 的工作空间，用户可以在 Worker 目录下放代码仓库。
- **前端功能偏基础**: 与 Multica 的完整 Issue 看板、Squad 团队、Skill 市场等功能相比，OpenBee 的 Web 控制台主要以监控和管理为主，缺乏高级协作功能。
- **无多租户**: 当前为单用户设计，没有组织/工作空间的概念。如果要在团队内部共享管理后台，需要额外的认证和隔离层。
- **测试覆盖**: 从项目结构看，缺乏明显的测试目录和测试文件。核心组件（Feeder、Gateway、Dispatcher）的单元测试对长期维护很重要。

### 与同类项目的定位差异总结

三个项目虽然都在做"AI Agent 协作平台"，但切入角度完全不同：

- **OpenBee** = "把 AI 当员工管"，从 IM 对话出发，强调 Worker 的组织身份和任务调度
- **Multica** = "把 AI 当同事管"，从 Issue 看板出发，强调 Agent 作为团队成员的协作流程
- **Leros** = "把 AI 当执行单元管"，从消息编排出发，强调可编程的工作流和分布式架构

如果你需要在 IM 群里@一个 AI 员工让它干活，选 OpenBee。如果你需要 Issue 驱动的开发流程自动化，选 Multica。如果你需要大规模分布式 Agent 编排，选 Leros。
