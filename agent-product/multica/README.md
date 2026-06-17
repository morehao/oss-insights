# Multica — 托管式 AI Agent 协作平台解读

> **GitHub:** [https://github.com/multica/multica](https://github.com/multica/multica)

## 第一部分：快速概览

### 这是什么

Multica（**Mul**tiplexed **I**nformation and **C**omputing **A**gent）是一个开源托管式 AI Agent 协作平台。它的核心理念是：**让 AI 编码 Agent 成为团队中的第一公民队友**——像给同事分配 Issue 一样，把任务指派给 Agent，Agent 自主执行、汇报进展、提交代码。

项目名称致敬 Multics（1960 年代的分时操作系统）——Multics 让多用户共享一台机器，Multica 让多个 Agent 和人类在同一个任务面板上协同工作。标语："你的下一个 10 个员工不会是人类。"

### 解决的痛点

当前使用 AI 编码助手（Claude Code、Codex、Copilot CLI 等）的典型问题：

- **手动搬运**：在终端和编辑器之间复制粘贴 prompt，丢失上下文
- **无持久化**：每次对话都是孤立的，没有任务历史、没有进度追踪
- **无团队协作**：Agent 的工作结果只有自己知道，队友无法查看状态或接力
- **无调度机制**：无法定时触发任务、无法通过外部事件（GitHub Issue、Webhook）自动启动工作
- **无能力沉淀**：每次解决问题的方法无法复用，团队经验无法累积

### 适用场景

- 2-10 人的 AI 原生工程团队
- 需要将重复性编码任务（bug 修复、小功能开发、代码审查）自动化
- 希望建立"AI 同事"协作流程的团队

### 核心能力清单

| 能力 | 说明 |
|------|------|
| **Issue 系统** | 完整的 Issue 管理（状态、优先级、标签、依赖、评论），Agent 可作为 assignee |
| **Agent 管理** | 注册 Agent、配置运行时、分配并发任务数、监控运行状态 |
| **Agent 模板** | 预配置模板，从模板创建 Agent 时自动导入技能和指令 |
| **Squad 团队** | 将多个 Agent（和人类）组成团队，由领导 Agent 路由任务 |
| **Autopilot 自动飞行** | 定时（Cron）、Webhook 或手动触发自动化工作流 |
| **Skill 技能库** | 可复用的 workflow，每次解决问题的方案可以沉淀为 Skill |
| **实时 WebSocket** | 任务进展、Agent 状态、聊天消息的实时推送 |
| **多运行时** | Agent 可以在本地（用户机器）或云端通过 Daemon 运行 |
| **多平台前端** | Web（Next.js）、Desktop（Electron）、Mobile（React Native） |
| **多语言界面** | 英文、简体中文、韩文、日文 |
| **GitHub 集成** | GitHub App Webhook 接收 Issue/PR 事件 |
| **飞书集成** | Lark Open Platform 集成，IM 消息收发、Bot Agent 路由 |

### 技术栈一览

| 层面 | 技术 |
|------|------|
| **后端语言** | Go 1.26+ |
| **HTTP 框架** | Chi |
| **WebSocket** | gorilla/websocket |
| **数据库** | PostgreSQL 17 + pgvector |
| **缓存 / 实时中继** | Redis（可选，多节点部署） |
| **数据层生成** | sqlc |
| **前端框架** | Next.js 16 (App Router) + React 19 |
| **桌面端** | Electron + electron-vite + react-router-dom |
| **移动端** | Expo / React Native |
| **组件库** | shadcn/ui on Base UI (Tailwind CSS v4) |
| **Monorepo** | Turborepo + pnpm workspaces |
| **服务端状态** | TanStack React Query |
| **客户端状态** | Zustand |
| **类型验证** | zod |

---

## 第二部分：架构深读

### 架构总览

```
┌──────────────────────────────────────────────────────────────────────┐
│                       Presentation Layer                             │
│  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────────┐   │
│  │ Next.js Web App │  │ Electron Desktop │  │ React Native      │   │
│  │ (App Router)    │  │ (Memory Router)  │  │ (Expo / RN)       │   │
│  └────────┬────────┘  └────────┬─────────┘  └─────────┬─────────┘   │
└───────────┼────────────────────┼──────────────────────┼──────────────┘
            │                    │                      │
            └─────────┬──────────┴──────────┬───────────┘
                      │                     │
             ┌────────▼─────────────────────▼──────────────────────────┐
             │                  Shared Logic Layer                     │
             │  ┌──────────────────┐  ┌────────────────────────────┐   │
             │  │ packages/core    │  │ packages/views             │   │
             │  │ (Zustand Stores, │  │ (Shared Business Pages)    │   │
             │  │  React Query,    │  │  - navigation adapter      │   │
             │  │  API Client)     │  │  - i18n / locales)         │   │
             │  └──────────────────┘  └────────────────────────────┘   │
             │  ┌──────────────────────────────────────────────────┐   │
             │  │ packages/ui (shadcn/Base UI, zero core imports)  │   │
             │  └──────────────────────────────────────────────────┘   │
             └──────────────────────┬──────────────────────────────────┘
                                    │ HTTP / WebSocket
             ┌──────────────────────▼──────────────────────────────────┐
             │                     Backend (Go)                        │
             │                                                        │
             │  Chi Router → Middleware → Handler → Service           │
             │                                                        │
             │  ┌────────────────┐  ┌────────────────┐               │
             │  │ Realtime       │  │ Daemon         │               │
             │  │ WebSocket Hub  │  │ WebSocket Hub  │               │
             │  └───────┬────────┘  └───────┬────────┘               │
             │          │                   │                         │
             │  ┌───────┴───────────────────┴──────────────────────┐  │
             │  │          Event Bus (in-process pub/sub)          │  │
             │  │   WS Broadcast / Activity Log / Notifications    │  │
             │  └───────────────────────┬──────────────────────────┘  │
             │                          │                             │
             │  ┌───────────────────────▼──────────────────────────┐  │
             │  │  Scheduler + Integrations                        │  │
             │  │  (DB-backed Cron / GitHub / Lark / Stripe)       │  │
             │  └──────────────────────────────────────────────────┘  │
             └──────────────────────────┬─────────────────────────────┘
                                        │
             ┌──────────────────────────▼─────────────────────────────┐
             │                     Data Layer                         │
             │  ┌─────────────────────────┐   ┌───────────────────┐   │
             │  │ PostgreSQL 17 + pgvector │   │ Redis (optional)  │   │
             │  │   - 主数据库             │   │   - 实时中继      │   │
             │  │   - 矢量存储             │   │   - 缓存 / 限流   │   │
             │  └─────────────────────────┘   └───────────────────┘   │
             └────────────────────────────────────────────────────────┘
                                        │ WebSocket
             ┌──────────────────────────▼─────────────────────────────┐
             │                 Agent Execution Layer                  │
             │                                                        │
             │         Agent Daemon (runs on user machine)            │
             │              ├── Claude Code CLI                       │
             │              ├── Codex CLI                             │
             │              └── (Future: cloud runtime)               │
             └────────────────────────────────────────────────────────┘
```

### 项目结构总览

```
multica/
├── server/                      # Go 后端
│   ├── cmd/
│   │   ├── server/              # API Server 入口
│   │   ├── multica/             # CLI 工具
│   │   └── migrate/             # 数据库迁移运行器
│   ├── internal/
│   │   ├── handler/             # HTTP 处理器
│   │   ├── service/             # 业务逻辑服务
│   │   ├── realtime/            # 浏览器 WebSocket Hub
│   │   ├── daemonws/            # Daemon WebSocket Hub
│   │   ├── events/              # 进程内事件总线
│   │   ├── middleware/          # 中间件链
│   │   ├── auth/                # 认证（JWT/PAT/DaemonToken）
│   │   ├── scheduler/           # 调度器
│   │   ├── integrations/        # Lark / GitHub 集成
│   │   └── ...
│   ├── pkg/db/                  # sqlc 生成代码
│   └── migrations/              # ~120 个 SQL 迁移
├── apps/
│   ├── web/                     # Next.js 前端
│   ├── desktop/                 # Electron 桌面端
│   └── mobile/                  # React Native 移动端
└── packages/
    ├── core/                    # 无头业务逻辑（Zustand/ReactQuery/API）
    ├── views/                   # 共享业务组件
    ├── ui/                      # 原子 UI 组件
    └── tsconfig/                # TypeScript 共享配置
```

### 启动流程

```
Server 启动（server/cmd/server/main.go）
  │
  ├── 1. 初始化 Logger，检查关键配置（JWT_SECRET、邮件后端）
  │
  ├── 2. 连接 PostgreSQL（pgxpool）
  │
  ├── 3. 创建进程内事件总线（events.Bus）
  │
  ├── 4. 创建 WebSocket Hub 和 Daemon WebSocket Hub
  │      ├── Realtime Hub — 用于浏览器客户端
  │      └── Daemon WebSocket Hub — 用于 Daemon Agent
  │
  ├── 5. 可选：初始化 Redis 中继（sharded / dual / legacy 模式）
  │
  ├── 6. 创建服务层（TaskService / IssueService / AutopilotService / EmailService）
  │
  ├── 7. 构建 Chi 路由（~1150 行路由注册）
  │      ├── 公共路由：/health /ws /auth/*
  │      ├── Daemon API：/api/daemon/*
  │      ├── 用户路由：/api/me /api/workspaces /api/invitations
  │      └── 工作空间路由：/api/issues /api/agents /api/squads ...
  │
  ├── 8. 启动后台 goroutine 集群
  │      ├── Runtime Sweeper — 标记过期运行时为 offline
  │      ├── Heartbeat Scheduler — Agent 心跳管理
  │      ├── Autopilot Scheduler — 定时触发 Autopilot
  │      ├── Autopilot Failure Monitor — 检测失败运行
  │      ├── DB Stats Logger — 连接池指标日志
  │      └── Lark Inbound Supervisor — 飞书消息监听
  │
  └── 9. 启动 HTTP Server，等待 SIGINT/SIGTERM
```

### 核心请求链路

我们追踪一个典型场景：**人在 Web 端创建一个 Issue 并分配给 Agent，Agent 在本地 Daemon 上接收并执行，执行结果实时回传**。

```mermaid
sequenceDiagram
    participant User as 用户（浏览器）
    participant Next as Next.js 前端
    participant Go as Go 后端
    participant DB as PostgreSQL
    participant WS as WebSocket Hub
    participant Daemon as Agent Daemon

    User->>Next: 创建 Issue（分配 Agent A）
    Next->>Go: POST /api/issues
    Go->>DB: INSERT issue (assignee_type=agent)
    Go->>DB: INSERT agent_task_queue (agent_id=A)
    Go->>Daemon: daemon:task_assigned (via Daemon WS)
    Go-->>WS: publish event (issue.created)
    WS-->>User: 实时更新 Issue 列表

    Daemon->>Daemon: Claim task
    Daemon->>Go: POST /api/daemon/claim-task
    Go->>DB: UPDATE task_queue (status=dispatched)
    Go->>Daemon: task details + workspace context

    Daemon->>Daemon: 启动 Agent 运行时（Claude Code / Codex）
    Daemon->>Go: POST /api/daemon/report-progress (streaming)
    Go->>DB: UPDATE issue (status=in_progress)
    Go-->>WS: publish event (task.progress)
    WS-->>User: 实时进度更新

    Daemon->>Go: POST /api/daemon/complete-task
    Go->>DB: UPDATE issue (status=done)
    Go->>DB: UPDATE task_queue (status=completed)
    Go-->>WS: publish event (task.completed)
    WS-->>User: Issue 状态更新 + 结果展示
```

### 模块职责划分

| 模块 | 路径（server/internal/） | 职责 |
|------|-------------------------|------|
| **Handler** | `handler/` | HTTP 请求处理，参数校验、响应组装 |
| **Service** | `service/` | 业务逻辑编排，事务协调 |
| **Realtime Hub** | `realtime/hub.go` | 浏览器 WebSocket 连接管理、订阅分发 |
| **Daemon Hub** | `daemonws/hub.go` | Daemon Agent 连接管理、心跳、任务分发 |
| **Event Bus** | `events/bus.go` | 进程内发布/订阅，驱动 WS 广播和后台处理 |
| **Middleware** | `middleware/` | 鉴权、工作空间解析、限流 |
| **Auth** | `auth/` | 多模式认证（JWT / PAT / Daemon Token） |
| **Scheduler** | `scheduler/` | DB 租赁式定时任务调度 |

---

## 第三部分：核心实现深度解析

### 数据模型设计

Multica 的数据模型围绕 **工作空间（Workspace）多租户** 和 **多态分配（Polymorphic Assignment）** 设计：

```mermaid
erDiagram
    user ||--o{ member : ""
    member }o--|| workspace : ""
    workspace ||--o{ agent : ""
    workspace ||--o{ issue : ""
    workspace ||--o{ squad : ""
    member ||--o{ agent : "owner"
    agent ||--o{ agent_task_queue : ""
    issue ||--o{ agent_task_queue : ""
    issue }o--o| agent : "assignee_type + assignee_id (多态)"

    user {
        uuid id PK
        string name
        string email
    }

    workspace {
        uuid id PK
        string name
        string slug
        jsonb settings
    }

    member {
        uuid user_id FK
        uuid workspace_id FK
        enum role "owner | admin | member"
    }

    agent {
        uuid id PK
        uuid workspace_id FK
        string name
        enum runtime_mode "local | cloud"
        enum status "idle | working | blocked | error | offline"
        int max_concurrent_tasks
        uuid owner_id FK
    }

    issue {
        uuid id PK
        uuid workspace_id FK
        string title
        enum status "backlog | todo | in_progress | in_review | done | blocked | cancelled"
        string assignee_type "member | agent"
        uuid assignee_id
        string creator_type "member | agent"
        uuid creator_id
        jsonb metadata
    }

    agent_task_queue {
        uuid id PK
        uuid agent_id FK
        uuid issue_id FK
        enum status "queued | dispatched | running | completed | failed | cancelled"
    }
```

**核心表设计要点：**

```sql
-- 多态分配：Issue 的 assignee 可以是 member 或 agent
-- server/migrations/xxx_init.up.sql
CREATE TABLE issue (
    id            UUID PRIMARY KEY,
    workspace_id  UUID NOT NULL REFERENCES workspace(id),
    title         TEXT NOT NULL,
    status        TEXT NOT NULL DEFAULT 'backlog',
        -- backlog | todo | in_progress | in_review | done | blocked | cancelled
    assignee_type TEXT,  -- 'member' | 'agent'
    assignee_id   UUID,
    creator_type  TEXT NOT NULL,  -- 'member' | 'agent'
    creator_id    UUID NOT NULL,
    ...
);

CREATE TABLE agent (
    id                  UUID PRIMARY KEY,
    workspace_id        UUID NOT NULL REFERENCES workspace(id),
    name                TEXT NOT NULL,
    runtime_mode        TEXT NOT NULL DEFAULT 'local',
        -- 'local' | 'cloud'
    status              TEXT NOT NULL DEFAULT 'idle',
        -- idle | working | blocked | error | offline
    max_concurrent_tasks INT DEFAULT 1,
    owner_id            UUID REFERENCES member(id),
    ...
);

CREATE TABLE agent_task_queue (
    id         UUID PRIMARY KEY,
    agent_id   UUID NOT NULL REFERENCES agent(id),
    issue_id   UUID NOT NULL REFERENCES issue(id),
    status     TEXT NOT NULL DEFAULT 'queued',
        -- queued | dispatched | running | completed | failed | cancelled
    ...
);

CREATE TABLE comment (
    id          UUID PRIMARY KEY,
    issue_id    UUID NOT NULL REFERENCES issue(id),
    author_type TEXT NOT NULL,  -- 'member' | 'agent'
    author_id   UUID NOT NULL,
    type        TEXT NOT NULL DEFAULT 'comment',
        -- comment | status_change | progress_update | system
    content     TEXT NOT NULL,
    ...
);
```

关键设计决策：

- **多态分配**：`issue.assignee_type + assignee_id` 的组合让 Agent 和人类成员可以出现在同一个字段中，前端根据类型渲染不同样式（Agent 用紫色背景 + 机器人图标）
- **任务队列**：`agent_task_queue` 作为 Issue 和 Agent 之间的调度层，支持排队、分发、运行、完成的全生命周期管理
- **审计日志**：`activity_log` 记录所有操作，支持按工作空间和 Issue 追溯
- **JSONB 元数据**：`issue.metadata`、`workspace.settings` 等字段使用 JSONB 存储，保持核心表稳定同时支持扩展

### 双 WebSocket 架构

Multica 维护两套独立的 WebSocket 系统，分别服务不同的客户端：

**Realtime Hub（`server/internal/realtime/hub.go`）：**

- 服务于浏览器客户端（Web 前端、Electron 渲染进程）
- 按工作空间（workspace-scoped）、任务（task-scoped）、聊天会话（chat-scoped）进行订阅隔离
- 认证方式：JWT / PAT / Cookie
- 支持 Redis 中继模式实现多节点广播

**Daemon WebSocket Hub（`server/internal/daemonws/hub.go`）：**

- 服务于 Daemon Agent（运行在用户机器上的守护进程）
- 独立协议：`daemon:heartbeat`、`daemon:task_assigned`、`daemon:task_claimed` 等
- 每个 Daemon 连接携带 `ClientIdentity`（daemon_id、user_id、workspace_id、runtime_ids）
- 事件去重：避免冗余唤醒
- 批量心跳处理：减少连接开销

```go
// server/internal/daemonws/hub.go: 身份结构
type ClientIdentity struct {
    DaemonID    string   `json:"daemon_id"`
    UserID      string   `json:"user_id"`
    WorkspaceID string   `json:"workspace_id"`
    RuntimeIDs  []string `json:"runtime_ids"`
}
```

两种 Hub 在事件总线上注册监听器，当业务层发布事件时，自动广播到相关的 WebSocket 客户端。

### 事件驱动总线

`server/internal/events/bus.go` 实现了一个进程内发布/订阅系统：

```go
// server/internal/events/bus.go: 事件定义
type Event struct {
    Type           EventType
    WorkspaceID    string
    Actor          Actor
    TaskID         *string
    ChatSessionID  *string
    Data           any
}

type Bus struct {
    mu      sync.RWMutex
    subs    map[EventType][]Handler
    ...
}

func (b *Bus) Publish(ctx context.Context, event Event) {
    b.mu.RLock()
    defer b.mu.RUnlock()

    for _, handler := range b.subs[event.Type] {
        handler(ctx, event)  // 同步调用
    }
}
```

事件总线是整个系统的**协调中枢**，它的角色包括：

- **WS 广播**：Realtime Hub 和 Daemon Hub 分别订阅事件，推送到对应客户端
- **活动日志**：`activity_log` 订阅者记录所有域操作
- **通知触发**：Inbox 订阅者创建通知项
- **Autopilot 触发**：某些事件可以作为 Autopilot 的触发器

事件总线是同步调用的（同一 goroutine 内），避免了异步消息队列的复杂性，但代价是事件处理不能阻塞（Handler 应尽快返回，耗时操作应启动新 goroutine）。

### 调度系统

`server/internal/scheduler/` 实现了一个基于数据库租赁（lease-based）的周期性任务调度器：

```sql
-- sys_cron_executions 表跟踪调度执行
CREATE TABLE sys_cron_executions (
    id          UUID PRIMARY KEY,
    job_type    TEXT NOT NULL,
    status      TEXT NOT NULL DEFAULT 'pending',
    lease_until TIMESTAMPTZ,
    last_run_at TIMESTAMPTZ,
    ...
);
```

当前调度任务：

| Job | 间隔 | 说明 |
|-----|------|------|
| `rollup_task_usage_hourly` | 每小时 | 聚合任务用量数据 |

分布式安全性：多个 Server 实例通过数据库行级锁竞争租赁（`UPDATE ... WHERE lease_until < NOW()`），确保同一时刻只有一个实例执行。

### 状态管理架构

前端采用严谨的**服务端状态 vs 客户端状态分离**策略：

```
TanStack React Query（服务端状态）
  ├── Issues / Users / Workspaces / Inbox Items
  ├── Agents / Runtimes / Skills / Projects
  ├── Labels / Chat Sessions / Autopilots
  └── 所有 query 以 wsId 为 key → 工作空间切换自动刷新

Zustand Stores（客户端状态）— packages/core/
  ├── auth/store.ts         — 认证状态
  ├── chat/store.ts         — 聊天会话状态
  └── */stores/
      ├── view-store.ts     — 视图模式、过滤器
      ├── selection-store.ts— 选中项
      ├── draft-store.ts    — 草稿
      └── ...
```

关键约定：

- **WebSocket 事件只触发 React Query 失效**，从不直接写入 Zustand Store
- **Zustand 只管理 UI 瞬态**（展开/折叠、过滤器值、滚动位置）
- **状态变更是单向的**：WS 收到事件 → 失效 Query → 自动重新获取 → UI 更新

### 导航适配器模式

为了解决共享组件在不同平台（Next.js File Router vs Electron Memory Router）中的导航问题，Multica 引入 `NavigationAdapter` 接口：

```typescript
// packages/views/navigation/adapter.ts
export interface NavigationAdapter {
    push: (path: string) => void;
    replace: (path: string) => void;
    back: () => void;
    pathname: string;
    searchParams: URLSearchParams;
}

// Web 实现：包装 next/navigation
// apps/web/components/web-providers.tsx
function WebNavigationProvider({ children }) {
    const router = useRouter();
    const pathname = usePathname();
    // 桥接到 NavigationAdapter 接口
}

// Desktop 实现：包装 react-router-dom
// apps/desktop/src/renderer/src/platform/navigation.tsx
function DesktopNavigationProvider({ children }) {
    const navigate = useNavigate();
    const location = useLocation();
    // 桥接到 NavigationAdapter 接口
}
```

**共享组件**（`packages/views/`）调用 `useNavigation().push()` 或 `<AppLink>`，不直接依赖任何路由库。每个平台提供自己的适配器实现。

### API 前向兼容策略

Electron 桌面应用作为二进制分发，可能比后端版本更旧或更新。前向兼容通过以下手段保证：

```typescript
// packages/core/api/client.ts: 安全解析
function parseWithFallback<T>(schema: z.ZodSchema<T>, data: unknown, fallback: T): T {
    const result = schema.safeParse(data);
    if (result.success) return result.data;

    // 记录解析失败但不崩溃
    console.warn('API response parse failed', result.error);
    return fallback;
}

// 使用示例
const issue = parseWithFallback(issueSchema, response.data, defaultIssue);
```

原则：
- **每个 API 响应都经过 zod 验证**，没有裸 `as` 类型断言
- **可选链操作**贯穿整个代码
- **枚举值降级**：遇到未知枚举值走 `default` 分支
- **API 字段增删**不会导致前端崩溃，只丢失对应功能

---

## 第四部分：部署与配置

### 依赖项

| 依赖 | 版本 | 说明 |
|------|------|------|
| PostgreSQL | 17+ | 主数据库，需安装 pgvector 扩展 |
| Redis | 可选 | 实时中继、缓存、限流（单节点部署无需） |
| Go | 1.26+ | 后端编译 |
| Node.js | 20+ | 前端构建 |
| pnpm | 最新 | 包管理 |

### 快速启动

```bash
# 安装依赖
make dev

# 启动后端（Go Server + 数据库迁移）
cd server && go run ./cmd/server

# 启动前端（开发模式）
cd apps/web && pnpm dev
```

### 关键配置项

| 配置 | 说明 |
|------|------|
| `JWT_SECRET` | JWT 签名密钥（生产环境必需） |
| `DATABASE_URL` | PostgreSQL 连接串 |
| `REDIS_URL` | Redis 连接串（可选） |
| `EMAIL_BACKEND` | 邮件发送后端 |
| `STORAGE_BACKEND` | 文件存储后端（S3 / 本地） |
| `GITHUB_APP_ID` | GitHub App 集成配置 |
| `LARK_APP_ID` | 飞书集成配置 |

---

## 第五部分：同类项目对比

### 定位对比

| 维度 | Multica | Leros | Dify | CrewAI |
|------|---------|-------|------|--------|
| **定位** | Agent 协作任务平台 | 企业级 AI 工作平台 | LLM 应用开发平台 | Agent 编排框架 |
| **核心抽象** | Issue + Agent + Squad | Project + Session + Worker | 工作流 + 知识库 | Agent + Task + Crew |
| **交付形态** | 自托管 SaaS | 自托管（Server + Worker） | 自托管 SaaS | Python 库 |
| **Agent 类型** | Claude Code / Codex CLI | Eino / Claude / Codex | 自定义 Agent | 自定义 Agent |
| **任务管理** | 完整 Issue 系统（看板 + 状态 + 优先级） | Session + Task | 对话 / 工作流 | 无 |
| **团队协作** | ✅ 多成员 + Agent 协同 | ✅ 多成员 + Agent 协同 | ✅ 多成员 | ❌ 单用户 |
| **实时推送** | WebSocket（双 Hub 架构） | WebSocket（基于 NATS） | WebSocket | ❌ 无 |
| **定时任务** | Autopilot（Cron / Webhook / 手动） | 无 | 工作流触发 | 无 |
| **技能沉淀** | Skill 系统 | Skill 系统 | 插件 / 工具 | 工具 |
| **前端** | Next.js + Electron + RN | Next.js + Electron | React | 无 |
| **消息队列** | 进程内事件总线 | NATS JetStream | Redis / Celery | 无 |
| **Go 后端** | Chi + sqlc + pgx | Gin + GORM | Python（Flask） | Python |
| **认证方式** | 邮箱验证码 / Google OAuth | GitHub OAuth / JWT | 邮箱 / OAuth | 无 |

### 架构对比

| 维度 | Multica | Leros |
|------|---------|-------|
| **Server-Worker 分离** | 否（单进程 + 外部 Daemon） | 是（Server / Worker 独立进程，NATS 通信） |
| **Daemon 协议** | WebSocket（独立 Hub） | WebSocket（Worker 注册） |
| **Agent 执行** | 外部 CLI（Claude Code / Codex） | 内置 Eino + 外部 CLI |
| **实时中继** | Redis（可选） | NATS（必需） |
| **数据库** | PostgreSQL + pgvector | SQLite / PG / MySQL / ClickHouse |
| **多节点** | Redis 中继（sharded streams） | NATS 原生支持 |
| **前端框架** | Next.js 16 App Router | Next.js |
| **包管理** | pnpm + Turborepo | 未公开 |
| **代码生成** | sqlc（数据库 -> Go 类型） | GORM（Go 结构体 -> 数据库） |
| **状态管理** | React Query + Zustand | 未公开 |

### 关键差异

1. **任务模型差异**：Multica 以 Issue 为核心，Agent 是 Issue 的 assignee——这是"Agent 作为第一公民"的具体体现。Leros 以 Session 为核心，消息是基本工作单元。两种模型反映了不同的使用场景：Issue 模型适合工程团队的任务跟踪，Session 模型适合持续对话的 AI 工作空间。

2. **架构复杂度**：Leros 采用四平面架构 + NATS 消息队列，Server 和 Worker 分离，天然支持分布式部署。Multica 采用单进程 + 外部 Daemon，架构更简洁，适合中小团队快速部署，但多节点扩容需要 Redis 中继。

3. **Agent 运行时**：Leros 内置了基于 CloudWeGo Eino 的原生 Agent 运行时（不依赖外部 CLI），Multica 依赖外部 CLI（Claude Code / Codex）。这反映了不同的设计哲学：Leros 追求"一站式"，Multica 追求"与现有工具链集成"。

4. **Skill 系统**：两者的 Skill 系统都支持将解决方案沉淀为可复用组件，但 Multica 的 Skill 以文件 + frontmatter 形式管理，支持从本地源导入；Leros 的 Skill 与 MCP 协议集成更深。

---

## 第六部分：总结与评价

### 最值得学习的设计决策

1. **严格的包边界分离**：`packages/core/`（零 react-dom、零 localStorage、零 process.env）、`packages/ui/`（零 core 引用）、`packages/views/`（零 next 引用）——这种边界约束在 monorepo 中强制了架构整洁度，每个包的可测试性和可替换性都得到保证。

2. **状态管理分离哲学**：React Query 管理服务端状态，Zustand 管理客户端瞬态，WebSocket 事件只触发 Query 失效从不直接写 Store。这个约定简单但有效，避免了前端状态管理中常见的"状态在不同地方不同步"问题。

3. **双 WebSocket 系统**：将浏览器客户端和 Daemon Agent 的 WebSocket 连接分开设计，各自的协议、认证方式、生命周期管理独立演化。浏览器使用 Realtime Hub 按工作空间订阅，Daemon 使用 Daemon Hub 按运行时绑定——职责清晰。

4. **导航适配器模式**：为 Next.js File Router、Electron Memory Router 等不同路由系统提供统一接口，让 `packages/views/` 中的业务组件完全与平台解耦。这是一个被低估但非常实用的抽象。

5. **API 前向兼容策略**：Electron 二进制分发带来的版本漂移问题，通过 `parseWithFallback()`、zod 验证、可选链、枚举降级等一整套防御性编程手段解决。对这些细节的重视是生产级软件的标志。

6. **事件总线作为协调中枢**：使用进程内发布/订阅替代消息队列，在单节点部署场景下减少了运维复杂度，同时保持了模块间的解耦。事件类型系统（携带 `WorkspaceID`、`Actor`、`TaskID`）让 Handler 可以精确过滤感兴趣的事件。

### 潜在改进空间

- **Server-Worker 分离**：当前架构中后端和 Agent Daemon 是紧耦合的（通过 WebSocket 直连），如果未来需要支持云端 Agent 运行时，可能需要类似 Leros 的 Server/Worker 分离方案
- **测试覆盖率**：从项目结构看，后端缺乏独立的测试目录（相比 Leros 有 `backend/tests/`），前端测试策略也未显式声明

### 项目信息

> **仓库:** [https://github.com/multica/multica](https://github.com/multica/multica)
