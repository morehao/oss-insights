# Multica — 多 Provider 隔离与记忆架构

> 本文档分析 Multica 的「记忆」与「运行时隔离」能力。与早期的「仅 Codex」分析不同，Multica 现已支持 **18 个 Agent 运行时**（见 `18-runtimes-compare.md`），记忆与隔离机制从单一 provider 专用策略演进为**逐 provider 适配的通用框架**。
>
> 权威来源：`server/internal/daemon/execenv/`（57 个文件，隔离与上下文注入）与 `server/pkg/agent/`（执行协议）。文档日期：2026-08-03 更新。

## 一、设计哲学（保持不变）

Multica 的记忆与隔离遵循一个核心原则：**显式优于隐式（Explicit > Implicit）+ 强隔离优先（Isolation First）**。

- 所有记忆通道都有 UI 或文件可见——没有隐藏的、不可审计的存储
- 每个 CLI 原生的自动记忆 / 全局配置 / 跨任务共享状态，都通过**逐 provider 的隔离机制**显式切断
- 隔离是**运行时保真（runtime fidelity）的最低要求**：不能让本地机器上一个宿主的全局配置、记忆或认证泄漏进 Multica 任务

## 二、4 层记忆架构（L1-L4，仍然成立）

| 层级 | 名称 | 粒度 | 生命周期 | 持久化位置 | 用户可见 |
|------|------|------|----------|-----------|----------|
| L1 | PriorSessionID | 会话级 | 跨连续任务 | `agent_task_queue.session_id`（DB 字段） | 否（底层透传） |
| L2 | 运行时上下文文件 | 任务级 | 单次任务 | 工作目录文件（标记块管理） | 是（文件可见 + 可编辑） |
| L3 | 技能 + issue_context.md | 问题级 | 问题生命周期 | 各 provider 原生技能目录 / `.agent_context/` | 是（文件可见，技能可管理） |
| L4 | Comments 增量指针 | 增量级 | 增量查询 | 运行时提示词（不持久化） | 否（仅指针提示） |

**L1 → L4 分层回退**的机制未变：L1 恢复对话连续性 → 不可用时 L2 注入任务身份 → L3 提供问题上下文 → L4 增量更新。

## 三、L2：运行时上下文注入（从「Codex 专用」到「18 种 provider」）

关键演进：早期只有 `CLAUDE.md`，现在按 provider 写入**不同的原生上下文文件**，通过 `runtime_config.go` 的 `runtimeConfigPath()` 集中映射（`server/internal/daemon/execenv/runtime_config.go:191`）：

| Provider | 目标文件 | 说明 |
|----------|---------|------|
| `claude` | `CLAUDE.md` | Claude Code 原生 |
| `codebuddy` | `CODEBUDDY.md` | CodeBuddy 的原生记忆文件（**不是** CLAUDE.md，见 `runtime_config.go:196`） |
| `qwen` | `QWEN.md` | Qwen Code 原生（避免跨运行时歧义） |
| `traecli` | `AGENTS.md` | **特例**：traecli 读 `.trae/rules/` 而非 AGENTS.md，故 brief 通过内联 SystemPrompt 交付，文件仅作 parity/visibility（`runtime_config.go:173` 的 `providerNeedsInlineSystemPrompt`） |
| 其余 13 个 | `AGENTS.md` | codex、copilot、opencode、deveco、openclaw、hermes、pi、cursor、kimi、kiro、antigravity、qoder、qoderclicn、grok |

### 标记块机制（未变）

`runtimeMarkerBegin/End`（`runtime_config.go:29`）用一个 HTML 注释标记块包住 Multica 注入内容，保证：

1. **保留用户内容**：`local_directory` 项目资源下不覆盖用户已有的 CLAUDE.md，而是追加标记块
2. **幂等更新**：重复运行只替换标记块内内容
3. **精确回滚**：`Cleanup` 区分「我们创建的文件」（无分隔符，直接删除）与「预先存在的文件」（有 `runtimeManagedSeparator`，剥离分隔符+块）——字节级还原（`runtime_config.go:33` 的 `runtimeManagedSeparator = "\n\n"`）

**注**：migration 120 的 protocol_family CHECK 里还有 `gemini`，但 `agent.go` 的 18 个 `SupportedTypes` 无 `gemini`——`runtime_config.go` 里也没有 GEMINI.md 分支（`gemini` 命中 default 返回 `""`）。GEMINI.md 是历史遗留的设计，尚无正式后端。

## 四、L3：技能写入的多 provider 原生目录

Multica 不再只写 `.claude/skills/`，而是把技能放到**每个 provider 原生发现的目录**（`runtime_config.go:159-175` 的注释），用 provider 自己的「技能发现机制」接管：

| Provider | 技能目录 |
|----------|---------|
| `claude` | `.claude/skills/` |
| `codebuddy` | `.codebuddy/skills/` |
| `codex` | 通过 `CODEX_HOME`（per-task home 内的 skills） |
| `copilot` | `.github/skills/` |
| `opencode` | `.opencode/skills/` |
| `deveco` | `.deveco/skills/` |
| `openclaw` | `{workDir}/skills/`（经 per-task `openclaw-config.json` pin workspace） |
| `hermes` | per-task `HERMES_HOME/skills/`（`hermes_home.go` 播种） |
| `pi` | `.pi/skills/` |
| `cursor` | `.cursor/skills/` |
| `qoder` | `.qoder/skills/` |
| `antigravity` | `.agents/skills/` |
| `kiro` / `kimi` / `grok` | 项目技能目录（原生发现） |

**为什么分散到各目录**：编写到 provider 的「家目录」或「发现路径」，agent 才能零额外配置加载。这反映了 Multica 的「与现有工具链集成」哲学——不强行统一技能路径，而是适配每个 CLI 的约定。

## 五、逐 Provider 隔离矩阵

这是本更新最核心的部分：**「仅 Codex 的 home 隔离」扩展为每个 provider 各自的原生隔离策略**。隔离的核心问题是：*CLI 从本地机器的全局配置、跨任务 home、认证缓存、记忆库中读取了什么？会不会把宿主项目上下文或上一次任务的残留拖进本次任务？*

| Provider | 隔离手段 | 关键文件 |
|----------|---------|---------|
| **codex** | **最全面**：per-task `CODEX_HOME`（sandbox 配置、记忆禁用、multi-agent 禁用、技能剥离、认证共享 via symlink） | `codex_sandbox.go`、`codex_memory.go`、`codex_multi_agent.go`、`codex_skill_strip.go`、`codex_home.go` |
| **openclaw** | 双模式（local/gateway）、shim 验证、gateway pin | `openclaw_shim.go`、`openclaw_config.go` |
| **hermes** | per-task `HERMES_HOME` 播种技能，不替换整个 home | `hermes_home.go` |
| **cursor** | 独立 MCP 配置注入 | `cursor_mcp.go` |
| **claude** | hooks 自愈、settings path 注入（`ClaudeSettingsPath`） | `claude_plugins.go`（daemon） |
| **codebuddy** | 原生 `CODEBUDDY.md` 记忆文件，避免 CLAUDE.md 歧义 | `runtime_config.go` |
| 通用 | `isolation_unix.go` / `isolation_windows.go`：进程级隔离（等价于 chroot/用户命名空间或 scoped 会话） | `isolation.go` |
| 通用 | `runtime_skill_policy.go`：技能可见性策略门控哪个 skill 对哪个 provider 可见 | `skill_visibility.go` |

### 5.1 Codex 的记忆与 home 隔离（核心实例）

`codex_home.go` 的 `prepareCodexHomeWithOpts` 在一条调用链上执行多个配置注入，所有管控块按固定顺序排列在文件顶部：

```go
ensureCodexSandboxConfig    → sandbox 块（限制网络/文件系统访问）
ensureCodexMultiAgentConfig → multi-agent 块（禁用原生多 agent）
ensureCodexMemoryConfig     → memory-feature 块 + memory-config 块（禁用原生记忆）
```

**记忆禁用三开关**（`codex_memory.go`，写进 per-task `config.toml` 的 manage 块）：

| 开关 | 含义 |
|------|------|
| `features.memories = false` | 禁用记忆特性入口 |
| `memories.generate_memories = false` | 禁写（不生成新记忆） |
| `memories.use_memories = false` | 禁读（不使用已有记忆） |

**TOML 安全适配**：严格模式下禁止重新定义已存在的表，所以代码区分「无 `[features]`/`[memories]` 表」（用根级虚线键）与「有表」（注入表内）。

**逃逸舱口**：`MULTICA_CODEX_MEMORY=1` 可重新启用原生记忆——为明确接受泄漏风险的用户提供。

**幂等性**：`TestEnsureCodexMemoryConfigIdempotent` 覆盖 5 种配置形态，保证重复运行字节一致。

### 5.2 为什么是「认证共享 + 配置隔离」的组合

以 Codex 为例（也适用于其他 provider）：`codex_home.go` 用 **symlink** 共享 `auth.json`（认证）和 `sessions/`（会话日志）到用户全局 `~/.codex/`，但用 **copy** 隔离 `config/config.toml/instructions.md`。这个「共享认证但隔离配置」的分工很关键：

- **认证要共享**：用户只需登录一次，CLI 复用其凭据
- **配置要隔离**：Multica 需要覆盖 sandbox / memory / multi-agent 行为，不能污染用户的全局配置
- **session 日志共享**：让会话历史可在 `~/.codex/` 查看，但同时配合 L1 的 per-task session 管理

### 5.3 OpenClaw 的双模式与 shim 校验

`openclaw_config.go` + `openclaw_shim.go` 覆盖两类问题：

1. **双模式**：`local`（守护进程在宿主机内嵌进程跑 agent 循环）vs `gateway`（经用户自定义 Gateway 或 per-task config pin 的端点路由）——`agent.go:75` 的 `OpenclawMode` 字段
2. **shim 校验**：Windows 的 npm 装包产生 `.cmd`/`.bat` shim，shim 本身可执行不代表其依赖的 `node` 可达（`openclaw_shim.go:28`）——需要额外链路检测，避免 daemon 以为 runtime 可用实则不可用

## 六、端到端数据流（更新版）

```
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
        ├─ gateResumeToReusedWorkdir（验证工作目录是否匹配，防泄漏）
        │
        ├─ execenv.Prepare / Reuse
        │     ├─ 创建 envRoot/{workdir, output, logs}
        │     ├─ writeContextFiles → .agent_context/issue_context.md + skills（各 provider 原生目录）
        │     ├─ [逐 provider 隔离] prepare*HomeWithOpts
        │     │     ├─ codex:   sandbox / memory / multi-agent / skill-strip（per-task CODEX_HOME）
        │     │     ├─ hermes:  per-task HERMES_HOME 播种技能
        │     │     ├─ openclaw: shim 校验 + local/gateway 双模式 + gateway pin
        │     │     └─ cursor:  MCP 配置注入
        │     └─ InjectRuntimeConfig → CLAUDE.md / CODEBUDDY.md / QWEN.md / AGENTS.md 标记块
        │
        └─ pkg/agent.New(provider).Execute（统一 Backend，见 18-runtimes-compare.md）
              │
              ▼
        Codex / Claude / Kimi / ...（18 个 provider CLI）
              │
              ├─ 读取运行时上下文文件（native 原生发现）
              ├─ 恢复会话（PriorSessionID → 对话历史）
              ├─ BuildNewCommentsHint → multica issue comment list（增量拉取）
              └─ 执行任务 + 返回 session_id + work_dir
                    │
                    ▼
              服务端 → 写回 agent_task_queue.session_id
```

## 七、代码文件清单（更新版）

| 文件 | 职责 |
|------|------|
| `execenv/runtime_config.go` | **核心**：L2 运行时上下文注入 + Cleanup 回滚，`runtimeConfigPath()` 集中映射 18 个 provider 的目标文件 |
| `execenv/runtime_config_kind.go` | 按任务类型（issue/chat/quick-create/autopilot）生成不同 brief——注意 issue 类型刻意不区分"分配触发"vs"评论触发"（MUL-5377，避免 prompt 缓存失效） |
| `execenv/runtime_skill_policy.go` + `skill_visibility.go` | L3：技能可见性策略，门控 per-provider |
| `execenv/context.go` | 技能目录写入（provider 原生路径） |
| `execenv/codex_*.go` | Codex 隔离全家桶：sandbox / home / memory / multi-agent / skill-strip / user-skills |
| `execenv/openclaw_*.go` | OpenClaw shim 校验 + config + gateway pin |
| `execenv/hermes_home.go` | Hermes per-task home 技能播种 |
| `execenv/cursor_mcp.go` | Cursor MCP 配置注入 |
| `execenv/reply_instructions.go` | L4：评论读取指针（热/温/冷三模式） |
| `execenv/isolation*.go` | 通用进程级隔离（unix/windows） |
| `execenv/sidecar_manifest.go` | sidecar 文件（issue_context.md）与 manifest 清理 |
| `pkg/agent/*.go` | 18 个 provider 的执行后端（见 `18-runtimes-compare.md`） |

## 八、演进总结与设计原则

**从「单一 Codex 策略」到「统一框架 + 逐 provider 适配」**：

1. **统一框架**：`Backend` 抽象 + `execenv` 生命周期（Prepare/Reuse/Cleanup）对所有 provider 一致
2. **逐 provider 适配**：上下文文件目标、技能目录、home 隔离方式、sandbox 粒度各不相同，各自独立演进
3. **显式 > 隐式**：所有记忆通道可见，CLI 原生隐藏记忆被显式禁用
4. **隔离优先**：每个 provider 都切断对宿主/跨任务状态的意外依赖，但**认证保持共享**（symlink），避免重复登录
5. **幂等安全**：所有配置注入保证第二次运行不修改文件

### 与早期分析的差异

- memory.md 原版聚焦「禁用 Codex 原生记忆」这一单一机制；本版把隔离/记忆推广为通用的多 provider 框架
- 「可用命令注入」已从 CLAUDE.md 单一目标扩展为 per-provider 原生文件（CODEBUDDY.md / QWEN.md / AGENTS.md）
- 新增了 OpenClaw / Hermes / Cursor 等 provider 的专用隔离策略，而不再只有 Codex
