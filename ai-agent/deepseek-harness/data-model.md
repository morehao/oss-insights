# 数据模型：事件日志的地面实现（持久化层）

> 子文档之一，聚焦 dsh 的持久化数据模型。覆盖事件日志持久化（`session-persistence`* 双后端）、非日志 KV 存储（`storage`* 可交换 backend + `storage-domain` 域表单）、派生投影缓存（`session-projection-cache`）。整体哲学见 [README](README.md)；本文上承 [session.md](session.md)（事件日志的落盘形态）、衔接 [deployment.md](deployment.md)（部署时的数据落地）。

核心论点：**这套「数据模型」不是一张 ER 图能讲完的业务表集合，而是两层可交换持久化的「事件溯源日志的地面实现」**——不可变事件日志是唯一事实源，KV 存储只是其上的投影底座；schema 演进被刻意做成「不迁移、只拒绝」。

## 根因：为什么持久化要长成两层可交换式

Dsh 的 Session 本质是一条不可变、append-only 的 `SessionEvent` 事件日志（内存态的事实源），见 `docs/subsystems/persistence.md` 首句[[docs/subsystems/persistence.md:5-5]()]()。这把"什么需要落地持久化"分成两个性质完全不同的集合，也因此长出两套独立的可交换 seam：

1. **事件日志本身**（真相）——必须按时间顺序、逐事件、连续 seq 地落地，崩溃后还能精确回放到"最后一个完整 turn"。它是追加写多、随机读少、需流式回放的数据。
2. **一切非事件的东西**（投影/键值状态）——工作区键值、投影缓存、各类域表。它们是"从真相推导出来的当前值"，改变频繁、需要按 key 随机读。

dsh 对这两类各给了一个服务抽象（`SessionPersistence` 与 `Storage`），又各配了可交换实现，这正是"Capability Seam 三件套"在持久化上的落地：接口（Service Definition）、提供方（backend）、消费方（domain 表单 / 投影缓存）。全部独立于 agent-loop 脊椎，无特权内核。

## 存储选型与理由

### JSONL：真相日志的"忠实回放"形态

事件日志后端以 append-only、不可变为核心。`session-persistence-jsonl` 把它实现为每会话一个 JSONL 逻辑日志，物理上用 zstd 压缩的拼接帧写入（编码名 `zstd`/`none`）[[packages/session/session-persistence-jsonl/src/format.ts:17-17]()]()。

选择 JSONL 的理由根植于"事件日志 = 可流式回放的真相"这一定位：

- **追加语义**天然贴合事件溯源：每次 append 只往文件尾写一行/一段，`appendLines` 用 `'a'` 打开、写后 `sync()` 即达持久化，崩溃后 `scanLog`/`SessionLogScanner` 按换行符增量切分、只解码完整记录，把"写一半"的尾部当 torn tail 容忍[[docs/subsystems/persistence.md:235-235]()]()。
- **流式回放**：回放不要求随机寻址，顺序扫一行解一行即可恢复，正好匹配真相日志"从头重演"的语义。
- **人可读与可压缩兼得**（`zstd` 关闭时）——一个会话一个文件、头部首行即 `SessionHeader`，`parseHeaderMeta` 只读首行即可做列表而不整日志解析[[packages/session/session-persistence-jsonl/src/format.ts:404-404]()]()。这让"会话文件就是事实源"在运维上可直接触达。

代价与权衡：没有 seek-by-seq，`readFrom(id, fromSeq)` 在 JSONL 上仍要解析整文件再跳过。这是真相日志甘愿付的代价——回放本来就要从头，按 seq 随机读不是它服务的场景。

### SQLite：一切可交换的投影/查询底座

事件日志的**派生读取**（投影、冷读、watermark 续读）需要真正按 seq 随机访问与按列过滤，这对 JSONL 是一处天然瓶颈。`session-persistence-sqlite` 用 `node:sqlite` 实现，但它早在 rc.8 起就**不再把每个 SessionEvent 映射成一事件一行**——见下节「Schema 17」的打包物理行。它提供 JSONL 给不了的 `readFrom` 尾读：`SELECT ... WHERE session_id = ? AND seq >= ?` 直接寻址，读规模随后缀而非日志总长[[packages/session/session-persistence-sqlite/src/index.ts:117-123]()]()。rc.8 的存储布局在 sqlite 子包内演进（`codec.ts`/`compression.ts`/`store.ts`），参考了 morlay 的 `session-persistence-rdb`，文档明确标注"统一关系库多后端设计 deferred、pre-release 不保证 schema 稳定"[[packages/session/session-persistence-sqlite/README.md:57-57]()]()。

所以"为什么 JSONL + SQLite 双形态"的答案不是性能攀比，而是**一个真相、两类读法**：JSONL 忠实、可回放、人可读（真相对外暴露）；SQLite 按 seq 投影、按 watermark 续读（对读侧派生）。二者共用同一套 `SessionPersistence` 抽象和同一套 `runPersistenceContract` 契约，且都通过共享的 `PersistenceCoordinator` 编排写路径，因此"底层怎么存"被完全封装在 seam 之下。

### 非日志 KV 存储：为何也要可交换

`storage` 子系统的存在则服务于"一切不是 SessionEvent 的持久化"。它同样是可交换的：hub `ctx.storage` 只是一个注册表 + 表单挂载点，本身不做任何 IO，backend 持有媒介，domain 表单持有语义[[packages/storage/storage/src/index.ts:3-3]()]()。

对应两个实现：

- `storage-json`：**一 unit 一个可读 JSON 文件**，`<root>/<unit>.json`，整个文件原子重写、人可读——"可读性就是它存在的理由"（format.ts 头注）[[packages/storage/storage-json/src/format.ts:1-1]()]()；适合低频改、需要人工审查/版本控制的整份状态（如 `session_projcache.json`、`workspace.json`）。
- `storage-sqlite`：**一库多 unit、每行一文档**（`key`/`value` JSON 文本），适合频繁更新、需要随机访问的数据[[docs/subsystems/storage.md:47-47]()]()。每个 unit 一张 `u_<unit>_<table>` 表 + `units`/`unit_globals` 元数据表[[packages/storage/storage-sqlite/src/unit.ts:1-1]()]()。

两条 backend 共享同一个 `KvFacet`/`KvUnit` 契约，并都通过 domain 表单（`ctx.storageDomain`）暴露给消费方。可交换的意义在这里与"不可变事件日志"哲学呼应：**真相（事件日志）与投影（KV 状态）分层，投影落在哪个媒介可随时换，但真相的语义不会因媒介而改变**。

## 实体关系总览

下面按"两类存储媒介 + 其上数据形态"给出关系图。事件日志是文档型/追加型，投影缓存与 KV 域是键值型，故用 `graph TB` 而非 `erDiagram`（键值/文档型实体用扁平有向图 + 配色分组表示三层归属）。

```mermaid
graph TB
    SE[SessionEvent 序列]
    SH[SessionHeader]
    COORD[PersistenceCoordinator]
    JSONL[jsonl 会话后端]
    SQL[sqlite 会话后端]
    B1[scanLog 整读回放]
    B2[readFrom 尾读]
    REG[ProjectionRegistry]
    CACHE[session_projcache 域]
    SNAPSHOT[ProjectionSnapshot]
    DOMAIN[storage-domain]
    JSONKV[json backend]
    SQLKV[sqlite backend]
    HUB[Storage hub ctx.storage]
    SE --> JSONL
    SE --> SQL
    SH --> JSONL
    SH --> SQL
    COORD --> JSONL
    COORD --> SQL
    JSONL --> B1
    SQL --> B2
    B1 --> REG
    B2 --> REG
    REG --> CACHE
    REG --> SNAPSHOT
    CACHE --- DOMAIN
    DOMAIN --> JSONKV
    DOMAIN --> SQLKV
    DOMAIN --- HUB
    classDef ev fill:#1e2a3a,stroke:#4db6ac;
    classDef pd fill:#2e1e3a,stroke:#ba68c8;
    classDef kv fill:#3a2e1e,stroke:#d4a04d;
    classDef br fill:#202020,stroke:#666;
    class SE,SH,JSONL,SQL,COORD ev;
    class REG,SNAPSHOT,CACHE pd;
    class DOMAIN,JSONKV,SQLKV,HUB kv;
    class B1,B2 br;
```

图例：青色组 = 事件日志真相层（session-persistence），紫色组 = 派生读取层（session-projection / projection-cache），褐色组 = 非日志存储层（storage），灰色 = 跨层桥接原语。

关系要点：

- 事件日志真相层（左）与派生读取层（中）之间靠 `PersistenceBackend.readFrom`（watermark 续读）和 `scanLog`/`load`（整读回放）两种桥接；
- 投影缓存是存储层（右）的消费方——它打开 `session_projcache` 域，经 domain 表单落到 json 或 sqlite backend；
- 图右即"一切非事件"的 KV 底座，domain 表单是唯一消费入口，`domain/changed` 事件在每次 durable 写后发出。

## 核心表 / 实体设计

下面按"真相日志 + 非日志 KV + 投影缓存"三大块拆表。字段表是更结构化的叙述，精确 DDL 以括号内行号为准。

### SessionEvent 邮箱信封（内存事实源）

职责：一条会话内不可变事件。信封是判别联合（type 与 data 的联合由编译器保证类型收窄），surface 字段为条件存在（仅 `user/message`/`assistant/message`/`tool/result` 三类型携带）[[packages/core/session/src/types.ts:336-336]()]()。

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| type | SessionEventType | 判别 key | 事件类型，`SessionEventMap` 可被插件合并扩展 |
| seq | number | 会话内单调连续 | append 的 contiguity 契约锚点 |
| time | number | Unix epoch ms | 事件时间戳 |
| data | SessionEventMap[K] | JSON-serializable | 负载，构造期强制 `isJsonValue` |
| ignorable | true(可选) | 仅日志型 | 未知类型的"可跳过"标记；缺省=必需，遇未知类型须拒绝而非静默丢弃 |
| sourceEventSeqs | number[](可选) | 仅 surface 事件 | 引用的先前事件 seq（assistant chunk→message、compaction 遮蔽） |
| surfaceOp | SurfaceOp(可选) | 仅 surface 事件 | 'append' 或 { op:'replace', start, end }，记录如何进 surface |

### SessionHeader（真相日志旁的元数据）

职责：**存储元数据，不是会话事件**——format version、cwd、lineage、seed 边界都对模型不可见，不进入 `deriveMessages()`，挂在 `session.header`[[packages/core/session/src/types.ts:61-61]()]()。

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| version | number | 写时从 SESSION_FORMAT_VERSION 盖章 | 后端加载时拒绝其它版本（不迁移） |
| id | SessionId | branded | 会话 id，与 Session 镜像 |
| createdAt | number | 非负 safe-integer epoch ms | 创建时刻 |
| cwd | string(可选) | 绝对路径 | 创建时工作目录 |
| parentSession | SessionId(可选) | | 被 fork 的来源，seed 谱系 |
| seedLength | number(可选) | | 继承的导引事件数，让 resume/replay 区分父史与子活 |
| origin | 'subagent'(可选) | | 由 subagent 子创建的产品粗分类（展示用） |
| delegationDepth | number(可选) | | 代理深度，持久化使递归预算跨重启存活 |
| agentPreset | string(可选) | deployment 按会话组合时置 | preset 决定工具的 prompt，持久化避免 resume 换组合重放 |

### JSONL 会话文件（真相物理形态）

职责：每会话一个物理文件，首行 `type:'session'` 头记录，随后的 JSONL 行或打包块为事件。路径布局 `root/<projectKey>/<encoded:sessionId>/session.jsonl[.zstd]`，sessionId 经 `encodeSegment` 单段安全编码[[packages/session/session-persistence-jsonl/src/format.ts:201-201]()]()。

| 位置 | 内容 | 说明 |
|------|------|------|
| 首行 | HeaderLine(`type:'session'` + header 字段) | 头即 storage 元数据；列表只读首行 |
| 后续行 | event 逐行 JSON 或 `packChunkRuns` 打包块 | 布局无关读：`scanLog` 恒按记录解码 |
| 物理编码 | zstd 拼接帧（默认）/ 明文 | 由 `JsonlCompression` 决定，后缀对应 |

### session-persistence-sqlite：`sessions` / `events` 表（rc.8 Schema 17 打包物理行）

**演进说明（rc.8）**：0.1.0-rc.8 把 SQLite 后端从「一事件一行、data 为 JSON 文本」重写为 **Schema 17 打包物理行**——见下。既有本条目的旧描述（events 行与 SessionEvent 1:1）已被本次重构取代。

职责：一个 SQLite 库承载全部会话，`sessions` 行是 out-of-log 元数据（≈SessionHeader）；`events` 行**不再是**与 SessionEvent 一一对应——普通事件仍是标量行，但可打包的连续 `assistant/chunk` 流被压成**打包物理行**。行存在性即物化信号：仅首次 append 写行，创建未 append 的会话无行、`list` 不可见[[packages/session/session-persistence-sqlite/src/schema.ts:26-26]()]()。

`sessions`：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | TEXT | PK | 会话 id |
| version | integer | NOT NULL | header 版本（事件词汇版本） |
| created_at / cwd / parent_session / seed_length / origin / delegation_depth / agent_preset | 多 | NULL=缺省可选 | 反映 SessionHeader 各可选字段 |
| incarnation | TEXT | NOT NULL，物化时 randomUUID | 日志持久化的稳定身份 |
| revision | INTEGER | NOT NULL，每次变更自增 | 单调日志变更 token，纯等值比较 |

`events`（DDL 列仍是这几列，但**类型与行含义**按 Schema 17 codec 变了）：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| session_id | TEXT | FK→sessions(id) ON DELETE CASCADE | 归属会话 |
| seq | INTEGER | 与 session_id 组成 PK | 标量行=该事件 seq；打包行=`seq0`（首个代表事件） |
| type | TEXT | NOT NULL | 标量行=逻辑事件类型；打包行=`text-chunks`/`reasoning-chunks`/`tool-call-chunks`（存储记录，非 `SessionEventMap` 成员） |
| time | INTEGER | NOT NULL | 标量行=事件时间；打包行=`time0`（首个代表事件） |
| data | ANY | NOT NULL | 标量行=JSON 文本；打包行=共享的打包 chunk 负载；≥4 KiB 时 zstd 压缩为 BLOB |
| source_event_seqs / surface_op | ANY/TEXT(可为空) | 条件 surface 元数据 | 打包行为 NULL；标量行 `source_event_seqs` 以 varint+ZigZag delta 编码为 BLOB |
| ignorable | INTEGER(可为空) | 0/1 | **打包行固定 0 作物理判别标记**；标量行 1 表示逻辑 ignorable 事件，否则 NULL |

关键是：**`type` 的 `*-chunks` 存储标签与"未知逻辑事件拒绝"不冲突**——decoder 先看 `ignorable` 是否 0 判定打包行，未知逻辑 ignorable 事件仍可复用存储标签名（`codec.ts:1-46`）。打包行各字段定义见 [[packages/session/session-persistence-sqlite/src/codec.ts:32-46]()]()。

每个 `packChunkRuns` 把**恰好连续、同块类型**的 delta 展开，未知字段/surface 元数据/seq 缺口/不安全 block 身份/不安全时间戳都保留为标量行；一行最多代表 **1 048 576 字节（`MAX_PACKED_DATA_BYTES`）** payload、**≥3 个成员（`MIN_PACKED_ROW_MEMBERS`）**，超长 run 被切分而不改变逻辑事件[[packages/session/session-persistence-sqlite/src/codec.ts:42-46]()]()。`appendBatch` 一次 `BEGIN IMMEDIATE` 事务里 `packChunkRuns(events)` 后逐记录 `bindRecord` 写入并只 increment revision 一次，旧逻辑事件行绝不重写[[packages/session/session-persistence-sqlite/src/store.ts:173-199]()]()。

### storage-sqlite：`units` / `unit_globals` / `u_<unit>_<table>`

职责：一库多 unit 的 KV 底座；记录值以 JSON 文本存 `value` 列，每原语单条 SQL（原子性交给 SQLite，无显式事务、无写队列）[[packages/storage/storage-sqlite/src/unit.ts:1-1]()]()。

| 表 | 字段 | 约束 | 说明 |
|----|------|------|------|
| units | name(unit/table) | (name,version) | unit 名 PK + 每 unit 版本盖章 |
| unit_globals | unit/value | PK→units, value TEXT | unit 全局单例槽 |
| u_<unit>_<table> | key/value | key PK, value TEXT NOT NULL | 每 unit 每 table 泛化记录表，value 为 JSON 文本 |

### storage-json：一 unit 单个文档

职责：整文件即 unit 的"当前净状态"，人读优先（pretty-print + 插入序稳定 key order），全局为 `null` 直到首写[[packages/storage/storage-json/src/format.ts:11-11]()]()。

| 字段 | 类型 | 说明 |
|------|------|------|
| unit.name / unit.version | 对象 | 文件头，标 unit 身份与版本 |
| global | any | `null`="从未写"哨兵 |
| tables | Record<tableName, Record<key,value>> | 各表记录；`null` 原型避免 `__proto__` 污染 |

写入为原子整文件替换（temp+fsync+rename+目录 fsync）[[packages/storage/storage-json/src/atomic.ts:24-24]()]()。

### session-projection-cache：`session_projcache` 域

职责：投影检查点持久化，`(sessionId, key, ver, seq, val)` 的 RFC 形态；`row 从不错误，只可能过期`——ver 不匹配即丢弃（从非迁移）[[packages/session/session-projection-cache/src/spec.ts:24-24]()]()。日志身份绑定（createdAt/cwd）防止"同 id 不同生命周期"的旧行通过水位检查[[packages/session/session-projection-cache/src/spec.ts:39-39]()]()。

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| createdAt / cwd | z.number/z.string.optional | checkpointIdentity | 日志身份（列表/冷读时对 live header 校验） |
| rows | Record<key, row> | rows 键为投影 key | 每 key 一个 checkpoint |
| row.ver | int≥0 | | 该 unit 的 stateVersion，不匹配即弃 |
| row.seq | int≥-1 | | 投影最后一次折叠的 seq（如何 stale） |
| row.val | z.json() | | 该投影 unit 的内部状态，纯 JSON |

写为整记录替换（whole-value 纪律）；域 spec `version:3` 升级即报废整个媒介[[packages/session/session-projection-cache/src/spec.ts:66-66]()]()。

### 附件对象库：`DSH_HOME/attachments/v1`（内容寻址二进制存储）

> **演进说明（rc.7）**：这是事件日志之外新增的**独立持久化面**——图片等二进制内容不进事件，字节落在内容寻址对象库，事件只存 `ImageAttachmentRef` 引用。完整能力面在 [attachments.md](attachments.md)。

职责：不可变二进制对象的持久化底座，与"事件日志存 ref、对象库存字节"严格分工。根目录 `DSH_HOME/attachments/v1`（版本化，跨版本对象格式互读靠目录切分），下分 `objects/<前两hex>/<sha256>` 内容寻址对象与 `tmp` 暂存区，`images` 对象以 `objects` 双层前缀摊开避免单目录过载。

其"不可变 + 防篡改"由双重机制保证（[attachments.md](attachments.md) 详述）：

- **内容寻址**：对象以字节 SHA-256 命名，`attachmentId` 形如 `sha256:...`，同一内容天然去重共享；`readImage` 回读时重算 digest 并逐项核对 `mediaType`/`bytes`/`width`/`height`，不匹配即报 `ATTACHMENT_CORRUPT`[[packages/attachment/attachment-local/src/store.ts:220-226]()]()。
- **崩溃安全**：写入走 `stage→write→sync→fsync目录→link→sync目录`，参考发布前同步 `objects` 目录，防"文件到目录项没到"的断电丢引用[[packages/attachment/attachment-local/src/store.ts:76-113]()]()、[[packages/attachment/attachment-local/src/store.ts:152-169]()]()。

| 字段 | 类型 / 约束 | 说明 |
|------|------------|------|
| `attachmentId` | `sha256:<64hex>`，不可变 | 内容寻址标识，非路径非 URL |
| `mediaType` | closed union `image/png/jpeg/webp/gif` | 从字节验出的格式 |
| `bytes`/`width`/`height` | number | 读回时逐项核对 |
| 对象路径 | `objects/<前2hex>/<sha256>` | 内容寻址摊开 |
| 暂存区 | `tmp/<uuid>` | 写入中暂存再 link 原子发布 |

**演进与 GC**：因恢复/分叉会话可能共享不可变对象，"没被引用即删"会误删仍在用的图，因此**对象 GC 与保留策略明确 deferred**（`readImage` 校验让"对象是否还在"可探测，但自动回收待引用感知 GC 落地[[packages/attachment/attachment/README.md:20-20]()]()）。v1 仅收四种栅格格式，音频/视频/通用文件留待各自生命周期契约[[packages/attachment/attachment/README.md:19-21]()]()。这不是数据模型妥协，而是「先保格式正确审计、后加媒体类型」的演进顺序。

## Schema 演进策略：刻意"不迁移，只拒绝"

dsh 的 schema 演进哲学与多数系统相反——**pre-release，无兼容承诺，宁可拒绝也不猜测迁移**。主要体现在几个单调版本号：

- `SESSION_FORMAT_VERSION = 0`：单一单调整数，无 major/minor 拆分[[packages/core/session/src/types.ts:56-56]()]()。决定是否 bump 看 **writer 发什么**，而非新 reader 能否接受——只有结构性变化才 bump（header 形态、Event 信封、core 事件语义、surface 机制/SurfaceEventType 集）；普通加事件类型不 bump，用 `ignorable` 防词汇增长[[packages/core/session/src/types.ts:40-40]()]()。
  - bump 的代价权衡见 constant 注释："near-identity 升级几乎免费，漏 bump 会让旧 runtime 静默读错新日志"——宁可过度拒绝也不静默重演被掏空的会话。
- 会话级版本：JSONL 后端从**首行 header** 就拒绝外来格式版本，`refuseForeignFormatVersion` 先于校验今日 header 形态与解码任何事件行，所以未来结构性格式仍报"升级方向"而非"corrupt"[[packages/session/session-persistence-jsonl/src/format.ts:240-240]()]()。SQLite 则先把整文件结构用自己 `SCHEMA_VERSION` pragma 挡住[[packages/session/session-persistence-sqlite/src/schema.ts:18-18]()]()。
- `SCHEMA_VERSION = 17`（SQLite 会话后端，**rc.8 自 15 bump 到 17**）：单调递增，只在前述会话持久化 DDL/codec 有破坏性变更时 bump 一档[[packages/session/session-persistence-sqlite/src/schema.ts:18-18]()]()。openDatabase 用 write-lock 持有下的 `PRAGMA user_version` 校验：缓存 `0`（全新）盖章为当前；非当前一律拒绝[[packages/session/session-persistence-sqlite/src/schema.ts:117-117]()]()。
- `STORAGE_SQLITE_SCHEMA_VERSION = 1`（KV sqlite）：`PRAGMA user_version`，与每 unit 版本正交；非当前拒绝，不迁移[[packages/storage/storage-sqlite/src/schema.ts:20-20]()]()。
- 域级版本：DomainSpec.version 被拒即拒绝打开[[packages/storage/storage-domain/src/spec.ts:35-35]()]()；投影 projcache 域 version 被拒即报废——cache 语义是"stale/不可读 cache 只花一次长尾重放，绝不产出错值"[[packages/session/session-projection-cache/src/spec.ts:62-62]()]()。

**回滚/灰度**：由于"拒绝不迁移"，上线前必须保证磁盘格式与代码版本配套；投影缓存依赖 `stateVersion` 的版本不匹配就地丢弃，是唯一"自动回滚"点（重读重放即可）。JSONL 改压缩编码需独立/全新 root 或选 legacy raw mode（见其 README）——因为格式版本无迁移。

## 读写路径

写路径有两条汇聚线，都以 `PersistenceCoordinator` 为编排中枢。下面以 JSONL 后端为例给出时序（SQLite 后端经由同一 coordinator，但把"文件 append"换成"一次事务内 `packChunkRuns` 后逐记录 insert"，见上文「Schema 17」）。

### 活会话写路径：session/event → write-behind 批量落地

```mermaid
sequenceDiagram
    participant S as Session
    participant C as Coordinator
    participant WB as WriteBehind
    participant B as JSONL backend

    S->>S: event 入日志（frozen）
    S->>C: session/event（installWritePath）
    C->>WB: enqueue（structuredClone 副本）
    WB->>WB: armTimer(maxDelayMs) 首事件定窗口
    WB-->>C: 后续事件只入队，不重置 deadline
    WB->>WB: onDeadline()
    WB->>B: write(batch)——appendBatch
    B->>B: appendLines('a', write, sync)
    B-->>WB: resolve（已持久化）
    Note over C,WB: 失败保留 pending；flush 立即重试
```

`installWritePath` 由 coordinator 构造时注册：`session/created` 记录头、`session/event` 拷贝冻结事件并入队、`session/flush` 作为即时持久化屏障、`session/disposed` 拉出 retirement 排水[[docs/subsystems/persistence.md:11-11]()]()。`session/event` 是同步通知，插件不阻塞生产者拷贝事件；`flush` 取消等待并排水至 quiescence[[packages/session/session-persistence/src/write-behind.ts:1-1]()]()。

seq contiguity 契约在 `appendCore` 强制：每个事件 seq 必须接续 `state.cursor`，`backend.appendBatch` 成功后 cursor/materialized 同步推进、invalidates 缓存[[docs/subsystems/persistence.md:296-296]()]()。

### 冷会话 / 冷读路径：load / inspect / readFrom

命中派生读取（投影缓存）时，从 SQLite 走 `loadStoredFrom` 尾读；JSONL 走 `loadStored`/`scanLog` 整读回放。恢复打断的 turn：backend 发现 `turn/start` 无 `turn/end`，**不截断**（单 turn 可能巨大），而是以合成 `turn/end { reason:{kind:'interrupted'} }` 收尾，不改变其前后任何独立事件[[docs/subsystems/persistence.md:15-15]()]()。仅当最后一条物理记录是残缺 fragment 时才丢弃 torn tail。

SQLite `loadStoredFrom`：

```mermaid
sequenceDiagram
    participant R as ReadModel
    participant P as SessionPersistence
    participant SQ as SQLite backend
    participant SC as scanRows

    R->>P: readFrom(id, fromSeq=floor)
    P->>SQ: loadStoredFrom(id, fromSeq)
    SQ->>SQ: SELECT… WHERE id=? AND seq>=?
    SQ->>SC: scanRows(rows, fromSeq)
    SC-->>SQ: preserved 前缀 + tornFrom
    SQ-->>P: {meta, events: preserved}
    P-->>R: 只回放 watermark 之后一段
```

关键语义：`readFrom` 返回的是**物理后缀**，`fromSeq >= 存储前缀` 时返回空事件而非报错；JSONL（顺序媒介）即使它仍解析整文件也保证只**返回/折叠** `seq>=fromSeq` 段——该原语限定"返回什么、折叠什么"，不限定每个 backend 的物理读[[docs/subsystems/persistence.md:344-344]()]()。

### 非日志 KV 写路径：domain 写 → backend 持久化 → domain/changed

```mermaid
sequenceDiagram
    participant D as DomainSpec 消费方
    participant T as KvTable
    participant B as KvUnit backend

    D->>T: table('x').put(key, value)
    T->>T: host.enqueue（单 per-domain 写链）
    T->>B: unit.putRecord(table, key, value)
    B->>B: 后端持久化（json 原子重写 / sqlite upsert）
    B-->>T: resolve（durable）
    T->>T: 然后 mutate 内存 records.set
    T->>T: 再 emit domain/changed(put)
    Note over T: 后端写失败 → 内存不动，读侧与媒介不背离
```

读写路径的全局连接：内存是权威（读同步自内存）、媒介先行持久化、事件在 durable 之后按写链顺序发出；`delete` 对缺席 key 返回 `false` 且不写不发事件；`update` 是写链槽位上的原子 read-modify-write[[docs/subsystems/persistence.md:98-98]()]()。

## 全局关联：数据模型是「不可变日志 + 可交换 seam」的物理载体

dsh 把「什么需要落地」拆成两类：事件日志（真相，需顺序 append + 崩溃回放）与投影/KV 状态（派生值，需随机读）。两层各配可交换 backend（JSONL 忠实回放 + SQLite 按 seq 投影；storage-json 人读 + storage-sqlite 频繁读写），由 `PersistenceCoordinator` 编排写路径。schema 演进「不迁移、只拒绝」——`SESSION_FORMAT_VERSION`/`SCHEMA_VERSION`/域级 version 一律单调盖章、非当前即拒，宁可过度拒绝也不静默重演被掏空的会话——这与其说是保守，不如说是「不可变事实源」哲学的自然后果：既然历史不可改，那演进就只能靠版本拒绝而非就地迁移。它把 [session.md](session.md) 的「Model-visible means logged」从内存典故落到磁盘形态，是 [architecture.md](architecture.md) 可交换后端哲学的又一实例。
