# OpenCode Session V2 持久化编排

> **关联文档:** [主解读](./README.md) · [系统架构深读](./architecture.md)

---

## 1. 设计动机

AI 编码 Agent 的会话（Session）本质上是人类与 AI 之间的多次交互。与 Chatbot 不同，编码 Agent 的交互包含文件读写、Shell 执行、工具调用，这些操作需要**精确的可恢复性**和**细粒度的执行控制**。

Session V2 的设计核心是三件事：

1. **崩溃恢复** — 执行到一半崩溃，重启后能精确恢复状态
2. **并发控制** — 同一 Session 不会同时跑两个 drain，多余的 wake 请求被合并
3. **上下文管理** — LLM 的系统提示由多个来源（环境、工具列表、Project 信息）拼装，需要高效的快照和增量更新

---

## 2. 核心模型

### 2.1 事件溯源架构

Session V2 采用**事件溯源（Event Sourcing）**，所有变更通过 EventV2.publish() 发布为持久化事件，Projector 将其投影到读模型表。

文件结构：

| 文件 | 路径 | 职责 |
|------|------|------|
| Session Service | `packages/core/src/session/session.ts` | 服务接口 + 实现 |
| Input 生命周期 | `packages/core/src/session/input.ts` | admit / promote / publish |
| SessionExecution | `packages/core/src/session/execution.ts` | 接口定义 |
| 本地执行 | `packages/core/src/session/execution/local.ts` | 本地实现 |
| Run Coordinator | `packages/core/src/session/run-coordinator.ts` | 执行协调器 |
| Runner | `packages/core/src/session/runner/` | 执行引擎 |
| Context Epoch | `packages/core/src/session/context-epoch.ts` | 系统上下文管理 |
| Projector | `packages/core/src/session/projector.ts` | 事件投影 |
| Compaction | `packages/core/src/session/compaction.ts` | 会话压缩 |
| Store | `packages/core/src/session/store.ts` | 读查询 |

### 2.2 关键数据结构

**SessionInput.Admitted** (`packages/schema/src/session-input.ts:14-23`):

```ts
const Admitted = Schema.Struct({
  admittedSeq: NonNegativeInt,    // 持久化序列号
  id: SessionMessage.ID,          // 消息 ID
  sessionID: SessionID,           // 所属会话
  prompt: Prompt,                 // 用户输入
  delivery: Delivery,             // "steer" | "queue"
  timeCreated: DateTimeUtcFromMillis,
  promotedSeq: NonNegativeInt.pipe(optional),  // 提升后的序列号
})
```

**SessionMessage 联合类型** (`packages/schema/src/session-message.ts:200-211`):

```ts
const Message = Schema.Union([
  AgentSwitched, ModelSwitched, User, Synthetic,
  System, Shell, Assistant, Compaction,
]).pipe(Schema.toTaggedUnion("type"))
```

**SessionExecution 接口** (`packages/core/src/session/execution.ts:9-18`):

```ts
interface Interface {
  active: Effect<ReadonlySet<SessionID>>
  resume: (sessionID) => Effect<void, RunError>
  wake: (sessionID) => Effect<void>
  interrupt: (sessionID) => Effect<void>
}
```

---

## 3. 生命周期

### 3.1 创建

```
SessionV2.create(input)
  ├─ store.get(sessionID) → 已存在则返回
  ├─ projects.resolve(directory) → 解析项目归属
  ├─ DB insert ProjectTable (onConflictDoNothing)
  ├─ 构建 SessionInfo: { id, slug, location }
  └─ events.publish(Created, { sessionID, info })
       └─ Projector: 写入 SessionTable
```

### 3.2 输入 (Prompt Admission)

**文件:** `packages/core/src/session/session.ts:360-385`

```
SessionV2.prompt({ id?, sessionID, prompt, delivery, resume?: true })
    │
    ├── 1. resolvePrompt(prompt)
    │       标准化文件附件、MIME 类型、Agent 附件
    │
    ├── 2. SessionInput.admit(db, events, { id, sessionID, prompt, delivery })
    │       ├─ find existing by ID → 已存在则返回 (幂等)
    │       └─ publish SessionEvent.PromptAdmitted
    │            └─ Projector: insert SessionInputTable
    │               (admitted_seq = durable seq, promoted_seq = NULL)
    │
    ├── 3. 冲突检测
    │       equivalent(admitted, expected) → 不匹配则 PromptConflictError
    │
    └── 4. execution.wake(sessionID)  (if resume !== false)
             调度执行
```

**关键设计: 幂等 admission** — 相同的 messageID 重复调用返回相同结果，这是重试安全的保证。

### 3.3 唤醒与合并 (Wake + Coordinator)

**文件:** `packages/core/src/session/run-coordinator.ts:67-103`

```
SessionExecution.wake(sessionID)
    │
    └── SessionRunCoordinator.wake(key)
        ├── 如果 Session 正在执行:
        │   entry.pendingWake = true     ← 仅设标志位，不新建 fiber
        │   return                         ← 完全合并
        │
        └── 如果 Session 空闲:
            create entry
            start drain fiber (force=false)  ← 仅检查 pending work
```

```
SessionExecution.resume(sessionID)
    │
    └── SessionRunCoordinator.run(key)
        ├── 如果正在执行:
        │   Deferred.await(entry.done)   ← 加入等待
        │
        ├── 如果正在停止 (等待中断完成):
        │   await done → 再次 run(key)   ← 重新执行
        │
        └── 如果空闲:
            create entry
            start drain fiber (force=true)   ← 强制执行
```

**设计要点:**
- `wake` 是"轻量通知"——同一个 Session 多个 `wake` 调用合并为一次 drain
- `resume` 是"加入执行"——无论 Session 什么状态，保证一次完整的执行
- `interrupt` 是"安全停止"——设置 stopping 标志，中断 fiber，清除 pendingWake

### 3.4 执行 (Drain)

**文件:** `packages/core/src/session/runner/llm.ts:383-406`

```
SessionRunner.run({ sessionID, force })
    │
    ├── 检查 pending work
    │   ├─ hasPending("steer")
    │   ├─ hasPending("queue")
    │   └─ !force && 无 pending → return (无工作可做)
    │
    ├── failInterruptedTools()  ← 清理被中断的工具执行
    │
    └── DRIVE LOOP:
        │
        while (shouldRun) {
            │
            └── while (needsContinuation) {
                    │
                    ├── 1. promote pending steers (最多到 cutoff)
                    │      SessionInput.promoteSteers(db, events, sessionID)
                    │      → publish SessionEvent.Prompted
                    │      → Projector: 更新 promoted_seq, 插入 user message
                    │
                    ├── 2. ContextEpoch.initialize() / prepare()
                    │      └── SystemContext.observe() → Generation
                    │           ├─ baseline: 完整渲染
                    │           ├─ reconcile: 检查是否变化
                    │           └─ replace: 完全替换基线
                    │
                    ├── 3. Model.resolve(session) → LLM Model
                    │
                    ├── 4. SessionHistory.entriesForRunner()
                    │      筛选: compaction 边界 + context epoch 基线
                    │
                    ├── 5. toLLMMessages(context, model) → LLM messages
                    │
                    ├── 6. compactIfNeeded()
                    │      估算 token → 超出窗口 → LLM 摘要压缩
                    │
                    ├── 7. llm.stream(request) → Stream<LLMEvent>
                    │      └─ 每个事件: publish via createLLMEventPublisher
                    │         → SessionEvent.Tool.* / Step.* / Text.*
                    │         → Projector: 写入 SessionMessageTable
                    │
                    ├── 8. Tool.settle() → 执行结果
                    │      需要继续? → needsContinuation = true
                    │
                    └── 9. 检查新的 steer 输入
                            hasPending("steer") → 继续内层循环
                }
                │
                └── hasPending("queue") → 继续外层循环
                    promoteNextQueued(db, events, sessionID)
        }
```

### 3.5 中断

```
SessionV2.interrupt(sessionID)
    │
    └── Coordinator.interrupt(key)
        ├── entry.stopping = true        ← 设置停止标志
        ├── entry.pendingWake = false    ← 清除待唤醒
        └── Fiber.interrupt(entry.owner) ← 中断执行 fiber
```

`interrupt` 对空闲或无此 Session 的调用是安全的（no-op），不报错。

---

## 4. 输入投递语义

Session V2 定义了**两种投递模式**，控制输入何时从"收件箱"变为"可见消息"：

### 4.1 Steer（转向）

```
Steer 输入 → admit (持久化 inbox)
    → 下一 provider-turn 边界 → promoteSteers()
        → publish SessionEvent.Prompted
        → Projector: User 类型消息写入 SessionMessageTable
        → 继续当前内层循环
```

- 同一 batch 的 steer 按 admitted_seq 顺序提升
- "下一 turn 边界"指的是当前 LLM 调用完成（无论是否产生工具调用）
- 处理 steer 后**重置** provider-turn allowance（Agent 可再次调用 LLM）

### 4.2 Queue（排队）

```
Queue 输入 → admit (持久化 inbox)
    → 当 Steering 完成后 → promoteNextQueued()
        → 仅提升 1 个 Queue 输入
        → 执行新的 drain 循环
```

- Queue 输入仅在 Session **没有工作可做时**才被提升
- 一次仅提升一个 Queue，提升后重新评估是否需要继续

### 4.3 对比

| 特性 | Steer | Queue |
|------|-------|-------|
| 提升时机 | 下一 turn 边界 | Session 将空闲时 |
| 批量处理 | ✅ 全部提升 | ❌ 一次一个 |
| 重置 Agent 轮次 | ✅ | ✅ |
| 典型场景 | 用户追问"改成这样" | 用户说"把这个也处理了" |

---

## 5. Context Epoch（系统上下文管理）

**文件:** `packages/core/src/session/context-epoch.ts`

System Context 是一组可观察的 `Source<A>` 集合，每个 Source 都有版本化渲染能力。

### 5.1 Source 定义

```ts
interface Source<A> {
  key: Key                          // "core/environment"
  codec: Schema.Codec<A>            // 序列化 Schema
  load: Effect<A | Unavailable>     // 获取当前值
  baseline: (current: A) => string  // 首次渲染
  update: (prev: A, curr: A) => string  // diff 渲染
  removed?: (prev: A) => string     // 移除通知
}
```

### 5.2 生命周期

```
ContextEpoch.initialize(db, events, context, sessionID)
    │
    ├── SystemContext.observe(value) → Generation
    │   观察所有 Source，渲染 baseline，生成快照
    │
    └── 存储: ContextEpochTable.insert({ sessionID, baseline, snapshot })
```

```
ContextEpoch.prepare(db, events, context, sessionID)
    │
    ├── 读取存储的 epoch
    ├── SystemContext.reconcile(value, snapshot)
    │
    ├── Unchanged → 直接返回 (无变化)
    ├── Updated → 更新快照 + 发布 ContextUpdated 事件
    └── ReplacementReady → 完全替换 epoch + 重写基线
```

### 5.3 初始化的三种结果

```ts
// 首次: 没有存储的 epoch → initialize()
//   ↓
// 之后: reconcile() 或 replace()
//   ├── Unchanged: 什么都不做
//   ├── Updated: 更新快照 → emit ContextUpdated event
//   └── ReplacementReady: 重写整个 epoch → compress + reload
```

### 5.4 内置 Source 示例

| Key | 说明 |
|-----|------|
| `core/environment` | 操作系统、运行时、Shell 信息 |
| `core/tools` | 当前 Agent 可用的工具列表 |
| `core/project` | 当前项目结构描述 |
| `core/preferences` | 用户配置偏好 |

---

## 6. Compaction（会话压缩）

**文件:** `packages/core/src/session/compaction.ts`

当上下文超出 LLM 窗口大小时自动触发：

```
1. 估算当前 msg 的 token 数 (超出窗口 → 触发)
2. 选取需要压缩的历史消息
3. LLM 压缩为 Compaction 摘要消息
4. 替换历史消息为摘要
5. 保留最近 keep.tokens 条消息作为 "recent" 上下文
```

Compaction 输出一种特定类型的 `SessionMessage`——`Compaction`，包含摘要文本和压缩参数。

---

## 7. 数据库模型

**文件:** `packages/core/src/session/sql.ts`

```
SessionTable
├── id: text PK
├── slug: text
├── location_id: text (FK → LocationTable)
└── created_at: integer

SessionMessageTable
├── seq: integer PK (auto-increment)
├── session_id: text (FK)
├── message: text (JSON — SessionMessage.Message)
└── created_at: integer

SessionInputTable
├── id: text PK
├── session_id: text (FK)
├── admitted_seq: integer (unique per session)
├── promoted_seq: integer? (NULL = not yet promoted)
├── delivery: text ("steer" | "queue")
└── prompt: text (JSON)

SessionContextEpochTable
├── id: text PK
├── session_id: text (FK)
├── baseline: text
├── snapshot: text (JSON — Source 序列化)
├── baseline_seq: integer (指向 SessionMessageTable)
└── created_at: integer
```

---

## 8. 设计评价

### 亮点

| 方面 | 评价 |
|------|------|
| **Admit/Promote 分离** | 输入先持久化再消费，天然支持重试和崩溃恢复 |
| **Steer/Queue 双语义** | 提供了一种"紧急插入"和"排队等候"的交互模式，Agent 交互更自然 |
| **Wake 合并** | 同一个 Session 的多次通知折叠为一次执行，避免并发问题 |
| **Context Epoch** | Source 的可观察+版本化设计，使系统上下文增量更新成为可能 |
| **事件溯源** | 所有状态变更可重放，支持审计和调试 |

### 与同类项目的对比

| 项目 | Session 模型 | 持久化 | 并发控制 | 上下文管理 |
|------|-------------|--------|---------|-----------|
| **OpenCode** | 事件溯源 + Admit/Promote | SQLite | Coordinator 串行 | Context Epoch |
| **Claude Code** | 会话快照 | 文件系统 | 无 (单线程) | 全量重新 |
| **Codex CLI** | 会话 ID | 数据库 | 无 | 全量重新 |
| **LangGraph** | Checkpoint + Node | 可插拔 (SQLite/Postgres) | 无 | 手动管理 |

### 潜在改进

- 目前所有执行在同进程完成，未来多进程/集群场景需要分布式协调
- Compaction 的触发策略比较保守（仅 token 超限时触发），可增加主动压缩选项
- Projector 处理 Event 是线性顺序的，高并发场景可能需要批量投影

---

## 9. 关键文件速查

| 文件 | 关键行 | 内容 |
|------|--------|------|
| `packages/core/src/session/session.ts` | 113-180 | Session Service 接口定义 |
| `packages/core/src/session/session.ts` | 360-385 | `prompt()` 实现 |
| `packages/core/src/session/input.ts` | 41-81 | `admit()` 幂等输入 |
| `packages/core/src/session/input.ts` | 245-266 | `promoteSteers()` |
| `packages/core/src/session/input.ts` | 268-287 | `promoteNextQueued()` |
| `packages/core/src/session/run-coordinator.ts` | 67-103 | Wake 合并 + Drain 调度 |
| `packages/core/src/session/runner/llm.ts` | 383-406 | Drain 主循环 |
| `packages/core/src/session/context-epoch.ts` | 40-78 | Context Epoch 生命周期 |
| `packages/core/src/session/compaction.ts` | — | 会话压缩算法 |
| `packages/schema/src/session-input.ts` | 14-23 | Admitted Schema |
| `packages/schema/src/session-message.ts` | 200-211 | Message 联合类型 |

---

> **关联文档:** [主解读](./README.md) · [系统架构深读](./architecture.md)
