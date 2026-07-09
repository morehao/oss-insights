# Tutti 记忆系统设计

> 分析时间: 2026-07-09

---

## 1. 设计哲学

tutti 采用 **Local-First** 记忆架构，所有用户状态存储在本机，不依赖网络。

核心决策：
- **单 SQLite 数据库**作为持久化的唯一真实源（Single Source of Truth）
- **事件流作为投影/通知**，不是持久化手段
- **不采用事件溯源（Event Sourcing）**，直接存储领域对象快照
- Go 守护进程是业务的**权威写入者**，桌面客户端只能发布"意图"

---

## 2. 存储层级

```
┌────────────────────────────────────────────┐
│  Layer 1: SQLite 数据库 (结构化数据)         │
│  ~/.tutti[-dev]/tuttid.db                   │
│  ┌────────────────────────────────────┐    │
│  │  tuttid_schema_migrations          │    │
│  │  workspaces, desktop_preferences   │    │
│  │  workspace_issues, workspace_apps  │    │
│  │  managed_model_provider_credentials│    │
│  ├────────────────────────────────────┤    │
│  │  agent_store (嵌入式模块)           │    │
│  │  workspace_agent_sessions          │    │
│  │  workspace_agent_messages          │    │
│  │  agent_targets                     │    │
│  └────────────────────────────────────┘    │
├────────────────────────────────────────────┤
│  Layer 2: 文件系统 (非结构化数据)           │
│  ~/.tutti/                                 │
│  ├── agent/runs/<session-id>/              │
│  ├── agent/attachments/<session-id>/       │
│  ├── apps/packages/<app-id>/               │
│  └── logs/                                 │
└────────────────────────────────────────────┘
```

---

## 3. SQLite 配置与连接

| 参数 | 值 | 说明 |
|------|-----|------|
| 驱动 | `modernc.org/sqlite` | 纯 Go 实现，无需 CGO |
| 最大连接数 | `1` | 单连接，避免并发写冲突 |
| `busy_timeout` | 5000ms | 等待锁超时 |
| `foreign_keys` | ON | 启用外键约束 |
| `journal_mode` | WAL | 写前日志，提升并发读性能 |

---

## 4. 核心表设计

### 4.1 Agent Session 表

```
workspace_agent_sessions
├── workspace_id              → 所属工作区
├── agent_session_id          → 全局唯一
├── origin                    → 创建源
├── agent_target_id           → 关联的 Agent 提供商
├── provider                  → 提供商类型
├── provider_session_id       → 提供商侧 ID
├── model                     → LLM 模型
├── settings_json             → JSON 配置
├── runtime_context_json      → JSON 运行时上下文
├── cwd                       → 工作目录
├── rail_section_kind         → 分类 (conversation/project/pinned)
├── rail_project_path         → 关联项目路径
├── rail_section_key          → 分类键
├── title                     → 会话标题
├── status                    → 状态
├── current_phase             → 当前阶段
├── last_error                → 最后错误
├── message_version           → 消息版本游标
├── last_event_at_unix_ms     → 最后事件时间
├── started_at_unix_ms
├── ended_at_unix_ms
├── pinned_at_unix_ms
├── created_at_unix_ms
├── updated_at_unix_ms
├── deleted_at_unix_ms        → 软删除
```

### 4.2 Agent Message 表

```
workspace_agent_messages
├── id                        → AUTOINCREMENT
├── agent_session_id          → FK
├── message_id                → 业务唯一标识（客户端提交 ID 派生）
├── version                   → 单调递增传输游标
├── turn_id                   → 所属 Turn
├── role                      → 角色
├── kind                      → 消息类型
├── status                    → 状态
├── payload_json              → 灵活负载
├── occurred_at_unix_ms
├── started_at_unix_ms
├── completed_at_unix_ms
├── created_at_unix_ms
├── updated_at_unix_ms
```

**关键设计**：使用 `UPSERT`（`ON CONFLICT DO UPDATE`）保证幂等写入。

---

## 5. 消息版本游标（ADR 0004）

```
session: message_version = 3
          ↓ 新消息写入
session: message_version = 4
          ↓ 消息行写入
message: { id: 42, message_id: "xxx", version: 4, ... }
```

- `message_version` 是 **per-session 单调递增计数器**
- 不是时间戳（解决了旧设计 timestamp-as-cursor 的问题）
- **加载相关**：counter 每次加载重新开始，`message_id` 才是稳定标识
- 消费者**永远不应**将 Version 视为全局稳定 key
- 展示顺序使用 `occurred_at_unix_ms`，Version 作为 tiebreak
- 自愈（Self-heal）= 从 `afterVersion=0` 完整重新同步

---

## 6. Session State Projection 模式

tutti 不使用简单的 UPSERT，而是通过投影函数做有状态条件更新：

```go
// services/tuttid/service/agent/activity_projection.go
projected := agentactivityprojection.ProjectSessionState(existing, hasExisting, input, now)
if !projected.Accepted {
    return false  // 拒绝旧事件
}
// 执行 UPSERT
```

投影规则：
- 如果 `input.OccurredAtUnixMS < session.LastEventUnixMS`（已发生更新的事件），拒绝
- 检查 status/phase 是否有实际变更
- 只有投影接受的状态变更才会持久化

---

## 7. Session 分类（Rail）

Agent 会话根据 `cwd` 和 `runtime_context` 自动分类：

| 分类 | 条件 | 展示 |
|------|------|------|
| conversations | 无项目关联 | 默认分类 |
| project | 匹配已知项目路径 | 按项目分组 |
| pinned | 用户固定 | 置顶分类 |

分类逻辑在 `classifyAgentSessionRailSectionTx` 中实现，通过 `userProjectPathsQuerier` 从 `user_projects` 表获取项目路径列表进行路径前缀匹配。

---

## 8. 双账本迁移

tutti 使用两个独立迁移账本：

```
tuttid_schema_migrations          → 主表迁移
  workspace_v1...v4
  workspace_issues_v1...v5
  desktop_preferences_v1...
  workspace_apps_v1...v3
  managed_credentials_v1
  app_factory_jobs_v1...v3

agent_store_schema_migrations     → Agent 存储迁移
  workspace_agent_activity_v1...v8
  agent_targets_v1
  workspace_agent_activity_rail_v1
```

迁移执行顺序：先主迁移，再 Agent 存储迁移。

**遗留兼容**：`claimLegacyMigrations` 机制将旧账本中已应用的 Agent 迁移记录复制到新账本。

---

## 9. 可嵌入的持久化模块

`packages/agent/store-sqlite` 被设计为**可嵌入模块**：

```
┌────────────────────────┐
│  宿主 (tuttid daemon)  │
│                        │
│  data/workspace/       │
│    sqlite_store.go     │ ← holds *sql.DB
│    agent_store.go      │ ← delegates to agentstore.Store
│                        │
│  biz/agentactivity/    │ ← type alias layer
│    activity.go         │   Repository = agentstore.Repository
│                        │   Session    = agentstore.Session
└────────┬───────────────┘
         │ 注入 *sql.DB + Options
         ▼
┌────────────────────────┐
│  packages/agent/       │
│    store-sqlite/       │
│      store.go          │ ← SQLite 实现
│      repository.go     │ ← 公共接口
│      migrations.go     │ ← 独立迁移账本
└────────────────────────┘
```

模块接受宿主注入：
- `*sql.DB` — 共享连接
- `Options` — `WorkspaceExists`、`ProjectPaths`、`NormalizeTarget` 等回调

这使得该模块可被不同宿主复用。

---

## 10. 软删除 vs 硬删除

| 操作 | 方式 | 原因 |
|------|------|------|
| 单个 Session 删除 | 软删除 (`deleted_at_unix_ms`) | 便于恢复和审计 |
| 单个 Message 删除 | 软删除 | 同上 |
| 删除 Workspace | 硬删除 (`DELETE`) | 终局操作，避免 FK 级联性能问题 |

新 Schema 去掉了 FK 约束，`ClearSessionsTx` 在同一事务中显式删除所有关联行。

---

## 11. 加密凭据

托管凭据（API Key 等）：**AES-256-GCM** 加密后存储在 `managed_model_provider_credentials` 表。加密密钥派生自机器本地密钥文件。

---

## 12. 文件系统布局

```
~/.tutti[-dev]/
├── tuttid.db                        → 主数据库
├── agent/
│   ├── sessions/<date>-<seq>/       → 工作目录
│   ├── runs/<session-id>/           → Sidecar 运行时
│   │   ├── sidecar-manifest.json
│   │   ├── codex-home/
│   │   └── tutti-agent-home/
│   └── attachments/<session-id>/    → 提示附件
├── agent-providers/
│   └── external-agent-registry/     → ACP 注册缓存
├── apps/
│   ├── packages/<app-id>/<version>/
│   └── installations/<app-id>/
│       ├── <scope>/runtime/
│       ├── <scope>/data/
│       └── <scope>/logs/
├── logs/
├── run/                             → 运行时元数据
└── bin/                             → CLI 入口
```

**设计约束**：文件系统路径禁止使用 `workspaceId` 作为目录段，Workspace 所有权只存在于 SQLite 中。

---

## 13. 数据流架构

```
[用户操作]
    │
    ▼
[Desktop Renderer]
    │ HTTP (REST API)
    ▼
[tuttid Daemon]
    │
    ├── api/ → service/ → data/ (SQLite)
    │                           ↓
    │                     eventstream (WebSocket)
    │                           ↓
    └──────────────────→ [Desktop Renderer]
```

- 持久化是"源"，事件流是"投影"
- 状态变更先写入 SQLite，再通过事件流通知 UI
- 桌面客户端不能发布最终的业务状态

---

## 14. 关键文档

- `docs/adr/0004-step4-hydration-monotonic-cursor.md` — 消息游标决策
- `docs/adr/0005-turn-lifecycle-optimal-projection.md` — Turn 生命周期投影
- `docs/adr/0008-turn-lifecycle-snapshot-authority.md` — 快照权威性
- `docs/conventions/local-state-storage.md` — 状态根目录约定
- `docs/architecture/business-event-stream.md` — 业务事件流
- `packages/agent/store-sqlite/` — 可嵌入持久化模块源码
- `services/tuttid/data/workspace/sqlite_store.go` — SQLite 实现
