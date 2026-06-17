# Multica 记忆（Memory）模块架构设计

> 本文档分析 Multica 项目中"记忆"能力的架构设计。Multica 采用去中心化、显式的分层记忆架构，没有独立的"记忆模块"数据库表或前端页面，而是通过 4 层正交机制组合出完整的记忆能力。

## 一、设计哲学

Multica 的记忆设计遵循一个核心原则：**显式优于隐式（Explicit > Implicit）**。

所有记忆通道都有明确的用户界面或文件可见——没有隐藏的、不可审计的存储。底层 CLI（Codex CLI）的原生自动记忆机制被**显式禁用**，因为它在 Multica 的控制范围之外引入了一个不透明的第二存储，可能导致跨任务和跨工作空间的上下文泄漏。

## 二、4 层记忆架构总览

| 层级 | 名称 | 粒度 | 生命周期 | 持久化位置 | 用户可见 |
|------|------|------|----------|-----------|----------|
| L1 | PriorSessionID | 会话级 | 跨连续任务 | `agent_task_queue.session_id`（DB 字段） | 否（底层透传） |
| L2 | CLAUDE.md / AGENTS.md | 任务级 | 单次任务 | 工作目录文件（标记块管理） | 是（文件可见 + 可编辑） |
| L3 | 技能 + issue_context.md | 问题级 | 问题生命周期 | `.claude/skills/` / `.agent_context/` | 是（文件可见，技能可管理） |
| L4 | Comments 增量指针 | 增量级 | 增量查询 | 运行时提示词（不持久化） | 否（仅指针提示） |

各层的关系是**分层回退（layered fallback）**：

```
L1 恢复对话连续性（同一 agent + issue 的历史）
  → 不可用时
L2 注入任务身份（who am I, what can I do）
  → L3 提供问题上下文（技能流程 + issue 元信息）
  → L4 增量更新（自上次以来发生了什么）
```

## 三、各层详细分析

### L1：PriorSessionID（会话级恢复）

**位置：** `server/internal/daemon/` — `types.go`、`daemon.go`、`handler/daemon.go`

#### 机制

同一 (agent, issue) 对的连续运行之间保持对话连续性：

```
任务完成 → CLI 返回 session_id
         → 存入 `agent_task_queue.session_id`
下次运行 → 查询 `GetLastTaskSession`（按 agent + issue + runtime 匹配）
         → 取 session_id → 传递给 `agent.Exec(--resume <session_id>)`
         → CLI 恢复上次对话的上下文
```

#### 守卫条件（防泄漏）

1. **runtime_id 匹配**：`GetLastTaskSession` 要求运行时 ID 必须一致，防止不同运行时（如 local vs cloud）的记忆交叉
2. **force_fresh_session**：`agent_task_queue.force_fresh_session` 为 true 时跳过恢复（用户手动要求重新运行）
3. **gateResumeToReusedWorkdir**（`daemon.go:2611`）：工作目录被 GC 回收后，会话 ID 被静默丢弃
4. **毒化会话排除**：排除因 `codex_semantic_inactivity` 等故障原因失败的会话，防止有毒上下文无限重放

#### DB Schema

```sql
-- agent_task_queue 表（迁移 020）
ALTER TABLE agent_task_queue ADD COLUMN session_id TEXT;
ALTER TABLE agent_task_queue ADD COLUMN work_dir TEXT;
ALTER TABLE agent_task_queue ADD COLUMN force_fresh_session BOOLEAN NOT NULL DEFAULT FALSE;
ALTER TABLE agent_task_queue ADD COLUMN chat_session_id UUID REFERENCES chat_session(id);

-- chat_session 表（迁移 033 + 060）
CREATE TABLE chat_session (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspace(id),
    agent_id UUID NOT NULL REFERENCES agent(id),
    creator_id UUID NOT NULL REFERENCES "user"(id),
    session_id TEXT,
    work_dir TEXT,
    runtime_id UUID REFERENCES agent_runtime(id),
    status TEXT NOT NULL DEFAULT 'active'
);
```

#### 代码路径

```
handler/daemon.go:claimTask → 查询 session_id
daemon.go:runTask → gateResumeToReusedWorkdir → agent.Exec(--resume <id>)
```

---

### L2：CLAUDE.md / AGENTS.md（运行时上下文注入）

**位置：** `execenv/runtime_config.go` — `InjectRuntimeConfig`、`CleanupRuntimeConfig`

#### 机制

每次任务运行时，Multica 向工作目录写入一个**标记块（marker block）**，注入代理需要知道的运行时上下文。支持 11 个 provider：

| Provider | 目标文件 |
|----------|---------|
| claude, codebuddy | `CLAUDE.md` |
| codex, copilot, opencode, openclaw, hermes, pi, cursor, kimi, kiro, antigravity | `AGENTS.md` |
| gemini | `GEMINI.md` |

#### 注入内容

```
# Multica Agent Runtime

## Agent Identity
**你是: GPT-Boy** (ID: `agent-uuid`)
[Agent Instructions]

## Requesting User
你代表 **Alice** 工作。They describe themselves as:
> Senior backend engineer

## Task Initiator
由 **Bob** (bob@co.com) 触发 — member of this workspace

## Workspace Context
[工作空间系统 prompt]

## Available Commands
multica issue get <id> --output json
multica issue create --title "..." [--description ...]
multica issue comment add <id> [--content ... | --content-stdin | --content-file ...]
...

## Repositories
github.com/org/repo

## Sub-issue Creation
...

## Issue Metadata
[语义化的 issue KV 存储读写规范]

## Skills
- **skill-name** — description

## Mentions
[mention 链接使用规范]

## Attachments
...

## Output
[输出规范]
```

#### 设计要点

1. **标记块隔离**：`<!-- BEGIN MULTICA-RUNTIME -->` / `<!-- END MULTICA-RUNTIME -->` HTML 注释标记，防止与用户自行维护的内容混淆
2. **幂等更新**：`writeRuntimeConfigFile` 同一目录多次运行只替换标记块内内容，不追加副本
3. **Cleanup 回滚**：`CleanupRuntimeConfig` 精确恢复文件到注入前的字节级状态：
   - 标记块前有 `runtimeManagedSeparator`（`\n\n`）→ 文件原本存在，移除此分隔符 + 标记块，写回剩余内容
   - 无分隔符且标记块是唯一内容 → 文件由 Inject 创建，直接删除
   - 无标记块 → no-op
4. **名称安全**：`sanitizeNameForBriefMarkdown` 转义用户名称中的 markdown 结构字符（`*`、`_`、`` ` ``、`\`、`[`、`]`、`<`）
5. **本地目录保护**：`local_directory` 流程下不会覆盖用户已有 CLAUDE.md，而是附加标记块 — Cleanup 后精确恢复

#### 示例：标记块注入

```markdown
# My Project's CLAUDE.md

Some user content here...
Conventions, preferences, etc.

<!-- BEGIN MULTICA-RUNTIME (auto-managed; do not edit) -->
# Multica Agent Runtime
## Agent Identity
...
## Available Commands
...
<!-- END MULTICA-RUNTIME -->
```

---

### L3：技能 + issue_context.md（问题级持久上下文）

**位置：** `execenv/context.go`、`execenv/execenv.go`

#### 机制

每次任务执行时，Multica 向工作目录写入两类持久化文件：

**a) 技能文件（Skills）**

写入 provider 原生的技能发现目录，agent 自动加载：

```
{workDir}/.claude/skills/issue-review/SKILL.md
{workDir}/.github/skills/bug-fixer/SKILL.md
{workDir}/.opencode/skills/deploy/SKILL.md
```

技能由用户在工作空间层面管理，是**可复用的 workflow 沉淀**——每次解决问题的方案可以提炼为技能。

**b) issue_context.md（`.agent_context/issue_context.md`）**

包含本次任务的问题 ID、触发器类型、新评论计数等运行时元信息。

**c) Codex 特殊处理**

Codex 运行时有独立的 `codex-home/skills/` 目录（因为 daemon 设置了 `CODEX_HOME` 为 per-task 目录以隔离配置）。`hydrateCodexSkills` 两阶段写入：

1. `seedUserCodexSkills` — 从共享 `~/.codex/skills/` 复制用户级技能
2. 写工作空间级技能（同名覆盖用户级）

写入前**清除整个 skills 目录**，防止复用路径上的残留。

#### 重用路径的清理

`Reuse()` 进入 `execenv.go:312` 时，两级清理确保资源不残留：

1. `removeReusedManagedSkillDirs` 回收平台自身写入的技能目录（即使 agent 在运行期间修改了文件）
2. `CleanupSidecars` 回滚剩余 sidecar 文件（`issue_context.md`、project resources）和 manifest

---

### L4：Comments 增量指针（增量级记忆）

**位置：** `execenv/reply_instructions.go`

#### 机制

agent 不直接从数据库拉取评论——而是通过提示词中的"指针"知道需要拉取多少新内容。三种模式，三段式回退：

```
BuildNewCommentsHint      → 热启动：有 since 锚点（上次运行的时间戳）
BuildResumedCommentsHint  → 温启动：无增量但有会话恢复
BuildColdCommentsHint     → 冷启动：全新启动，无 prior session
```

#### 热启动示例

```
3 条新评论，先读触发线程：
  multica issue comment list MUL-123 --thread <id> --since 2025-01-01T00:00:00Z
需要其他线程再全量：
  multica issue comment list MUL-123 --since 2025-01-01T00:00:00Z
```

#### 温启动示例

```
恢复会话，触发评论已注入。无其他新评论。
先用 --thread 拉取触发对话：
  multica issue comment list MUL-123 --thread <id> --tail 30
不要仅依赖恢复的会话记忆。
```

#### 冷启动示例

```
先读触发对话：
  multica issue comment list MUL-123 --thread <id> --tail 30
需要跨线程背景？
  multica issue comment list MUL-123 --recent 20
```

#### 设计要点

- **只传计数和游标，不传评论体**——服务端开销极低
- **引导先读触发线程**，而非盲目拉全量——减少 token 消耗
- **增量和评论体分离**：CLI 负责拉取，提示词只负责告知位置
- **复用 guard**：`BuildResumedCommentsHint` 显式指示「不要仅依赖恢复的会话记忆」，要求先拉取最新上下文

---

## 四、核心设计：Codex 原生记忆禁用机制

**位置：** `execenv/codex_memory.go`

### 问题背景

Codex CLI 内置了一个**隐藏的自动记忆子系统**：

1. 自动将 agent 轮次的摘要写入 `$CODEX_HOME/memories/raw_memories.md` + `state_*.sqlite`
2. 下次轮次自动读回，注入模型上下文
3. 对 Multica 用户**完全不可见**——从任何 Multica UI 都无法审计或编辑

这引入两个泄漏路径：

- **跨任务泄漏**：per-task `codex-home/memories/` 在 `Reuse()` 时保留，前次轮次的记忆自动喂给下次运行
- **跨工作空间泄漏**（issue #3130）：Codex CLI 可能读取 `~/.codex/memories/`（用户全局），将宿主项目的上下文拖入 Multica 任务——报告显示 `D:\Project\MoHaYu\WowChat` 的 Raw Memories 被注入一个全新的 Multica issue 的第一次 Codex 轮次

### 解决方案

`ensureCodexMemoryConfig` 向每个任务的 `config.toml` 写入**守护进程管理的标记块**：

```toml
# BEGIN multica-managed memory-feature (do not edit; regenerated by daemon)
memories = false
# END multica-managed memory-feature

# BEGIN multica-managed memory-config (do not edit; regenerated by daemon)
generate_memories = false
use_memories = false
# END multica-managed memory-config
```

两个独立块，覆盖三个开关：

| 开关 | 含义 |
|------|------|
| `features.memories = false` | 禁用记忆特性入口 |
| `memories.generate_memories = false` | 禁写（不生成新记忆） |
| `memories.use_memories = false` | 禁读（不使用已有记忆） |

Codex 读取这份 `config.toml` 后，既不会写也不会读记忆子系统，双泄漏路径同时切断。

### TOML 安全适配

TOML 严格模式下**禁止重新定义已存在的表**。代码需要适配用户已有配置的两种形态：

| 用户配置形态 | memory-feature 写法 | memory-config 写法 |
|-------------|--------------------|--------------------|
| 无 `[features]` / `[memories]` 表 | `features.memories = false`（根级虚线键） | `memories.generate_memories = false`（根级虚线键） |
| 有 `[features]` / `[memories]` 表 | 注入 `[features]` 表内：`memories = false` | 注入 `[memories]` 表内：`generate_memories = false` |

注入逻辑：

```go
// 简化逻辑
if hasFeaturesTable {
    injectInsideFeaturesTable("memories = false")
} else {
    prependRoot("features.memories = false")
}
if hasMemoriesTable {
    injectInsideMemoriesTable("generate_memories = false\nuse_memories = false")
} else {
    prependRoot("memories.generate_memories = false\nmemories.use_memories = false")
}
```

### 逃逸舱口

`MULTICA_CODEX_MEMORY=1` 环境变量可重新启用——为明确接受泄漏风险的用户提供。任何 truthy 值（`1`、`true`、`yes`、`on`，大小写不敏感）保留原生记忆；未设置或设置 falsy 值则禁用。

### 用户指令清理

`stripUserMemoryDirectives` 在写入管理块前，删除用户配置中冲突的自定义指令。覆盖的指令形态：

| 位置 | 匹配模式 |
|------|---------|
| TOML 根级 | `features.memories = ...`、`memories.generate_memories = ...`、`memories.use_memories = ...` |
| `[features]` 表内 | `memories = ...` |
| `[memories]` 表内 | `generate_memories = ...`、`use_memories = ...` |

TOML 子表（如 `[features.experimental]`、`[memories.advanced]`）不受影响——它们在自己作用域内不重新定义管理块设置的键。

### 幂等性保证

重复运行产生完全相同的文件。代码通过以下机制保证：

1. 先通过正则 `memoryFeatureBlockRe` / `memoryConfigBlockRe` 剥离已有管理块
2. 检测内容是否与原始数据相同，相同则跳过 `WriteFile`
3. `injectAfterHeader` 规范化块后空白行，确保剥离-重写后字节一致

测试验证：`TestEnsureCodexMemoryConfigIdempotent` 覆盖 5 种配置形态（根级键、`[features]` 表、`[memories]` 表、双表、空文件）的幂等性。

### 与其他系统的共存

`prepareCodexHomeWithOpts` 在一条调用链上执行三个配置注入，所有管理块在文件顶部按固定顺序排列：

```go
ensureCodexSandboxConfig   → sandbox 块
ensureCodexMultiAgentConfig → multi-agent 块
ensureCodexMemoryConfig    → memory-feature 块 + memory-config 块
```

`TestEnsureCodexMemoryConfigCoexistsWithSandboxAndMultiAgent` 验证三者共存时 TOML 依然正确解析。

## 五、端到端数据流

```
用户操作（评论 / 分配 / 聊天消息 / autopilot）
        │
        ▼
服务端 handler/daemon.go → claimTask
        │
        ├─ 查找 PriorSessionID（GetLastTaskSession / GetChatSession）
        ├─ 计算 NewCommentCount / NewCommentsSince
        ├─ 获取 workspace context / agent instructions / skills
        └─ 构建响应（含 PriorSessionID + work_dir + 上下文元数据）
        │
        ▼
守护进程 daemon.go → runTask
        │
        ├─ gateResumeToReusedWorkdir（验证工作目录是否匹配）
        │
        ├─ execenv.Prepare / Reuse
        │     ├─ 创建 envRoot/{workdir, output, logs}
        │     ├─ writeContextFiles → .agent_context/issue_context.md + skills/
        │     ├─ [codex] prepareCodexHomeWithOpts
        │     │     ├─ symlink sessions/（会话日志共享到 ~/.codex/）
        │     │     ├─ symlink auth.json（认证令牌共享到 ~/.codex/）
        │     │     ├─ copy config.json + config.toml + instructions.md（配置隔离）
        │     │     ├─ sanitizeCopiedCodexConfig（移除 [[skills.config]] 条目）
        │     │     ├─ ensureCodexSandboxConfig      → 限制网络 / 文件系统访问
        │     │     ├─ ensureCodexMultiAgentConfig    → 禁用原生多 agent
        │     │     └─ ensureCodexMemoryConfig        → 禁用原生记忆
        │     └─ InjectRuntimeConfig → CLAUDE.md / AGENTS.md 标记块
        │
        └─ agent.Exec（--resume <PriorSessionID>）
              │
              ▼
        Codex / Claude / 其他 CLI
              │
              ├─ 读取 CLAUDE.md（运行时上下文 + 可用命令 + 技能列表）
              ├─ 恢复会话（PriorSessionID → 对话历史）
              ├─ BuildNewCommentsHint → multica issue comment list（增量拉取）
              └─ 执行任务 + 返回 session_id + work_dir
                    │
                    ▼
              服务端 → 写回 agent_task_queue.session_id
```

## 六、核心文件清单

| 文件 | 职责 |
|------|------|
| `execenv/codex_memory.go` | **核心**：禁用 Codex CLI 原生自动记忆，写入 manage block 到 config.toml |
| `execenv/codex_memory_test.go` | 607 行全面测试：15 种场景，含幂等性、TOML 有效性、逃逸舱口 |
| `execenv/runtime_config.go` | L2：注入运行时上下文到 CLAUDE.md / AGENTS.md / GEMINI.md，含 Cleanup 回滚 |
| `execenv/reply_instructions.go` | L4：构建评论读取指针和回复指令（热 / 温 / 冷三种模式） |
| `execenv/execenv.go` | 执行环境准备：Prepare / Reuse / Cleanup，含技能写入和环境隔离 |
| `execenv/codex_home.go` | per-task CODEX_HOME 准备：symlink/copy 配置、调用所有 ensure* 函数 |
| `execenv/context.go` | 技能目录写入（provider 原生路径） |
| `daemon/daemon.go` | PriorSessionID 守卫（gateResumeToReusedWorkdir） |
| `daemon/types.go` | `Task.PriorSessionID` 定义 |
| `daemon/prompt.go` | 构建提示词时使用 PriorSessionID 和评论指针 |
| `handler/daemon.go` | 服务端 claimTask 会话查找逻辑 |
| `migrations/020_task_session.up.sql` | `agent_task_queue` 添加 session_id、work_dir |
| `migrations/033_chat.up.sql` | `chat_session`、`chat_message` 表 |
| `migrations/060_chat_session_runtime_id.up.sql` | `chat_session.runtime_id` |
| `migrations/066_force_fresh_session.up.sql` | `force_fresh_session` 字段 |

## 七、未来规划

`codex_memory.go:47-48` 原文：

> The long-term answer for durable agent context is a Multica-owned, user-visible, project- or issue-scoped memory store — not re-enabling Codex's hidden auto-memory.

Multica 团队有明确计划构建第一方记忆模块，而非重新启用 Codex 的隐藏记忆。未来记忆模块的约束条件：

1. **用户可见** — 可在 Multica UI 中审计和编辑
2. **作用域受控** — project 或 issue 级别，不会跨工作空间泄漏
3. **守护进程可控** — 写入和读取都经过服务端协调，而非 CLI 自主行为
4. **不绑定 Provider** — 不依赖任何特定 CLI 的原生记忆实现

当前 4 层去中心化架构是达成这一目标的过渡方案，每层的可替换性为未来统一记忆服务保留了接入点。

## 八、设计原则总结

1. **显式 > 隐式**：所有记忆通道都有明确的用户界面或文件可见，无隐藏存储
2. **分层回退**：L1→L2→L3→L4 逐层回退，每层覆盖上一层无法触及的场景
3. **禁用隐藏记忆**：Codex 原生记忆被显式禁用，防泄漏是第一优先级
4. **运行时隔离**：runtime_id 守卫 + 工作目录 GC 守卫 + 毒化会话排除
5. **增量审计**：Comments 增量指针只传计数，不传内容，评论体按需拉取
6. **幂等安全**：所有配置注入（memory / sandbox / multi-agent）保证第二次运行不修改文件
