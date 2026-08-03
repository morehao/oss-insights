# Multica — 18 种 Agent 运行时深度对比

> 本文档分析 Multica 的 Agent 运行时抽象层，对比其支持的 18 种编程 Agent CLI。配套阅读：`README.md`（整体架构）、`memory.md`（多 Provider 隔离与记忆）。
>
> 权威来源：`server/pkg/agent/agent.go` 的 `SupportedTypes` 白名单（`server/pkg/agent/agent.go:237`）。

## 一、统一抽象：Agent Backend

从最早的 "仅 Claude Code + Codex CLI" 演进，Multica 现在通过一个统一的 `Backend` 接口抽象所有编程 Agent CLI。这是 8 月分析时最大的结构性变化——运行时层从一个硬编码的二选一 switch，变成了可枚举、可扩展、每个 provider 独立实现的插件式架构。

```go
// server/pkg/agent/agent.go: Backend 统一接口
type Backend interface {
    // Execute 运行一个 prompt，返回流式输出的 Session
    Execute(ctx context.Context, prompt string, opts ExecOptions) (*Session, error)
}
```

核心抽象：

| 概念 | 说明 |
|------|------|
| **`Backend`** | 统一执行接口，一个 provider 一个实现 |
| **`Session`** | 流式结果封装：`Messages` 通道持续输出事件，`Result` 通道返回最终结果 |
| **`Message`** | 统一事件类型（text / thinking / tool-use / tool-result / status / error / log） |
| **`Result`** | 最终结果（status / output / usage / session_id / resume_rejected） |
| **`ExecOptions`** | 跨运行时通用参数（超时、重试、MCP 配置、thinking level、service tier 等） |
| **`TokenUsage`** | 带 `CostUSDTicks` 精准成本（xAI 的 1e-10 美元精度计量，见 `agent.go:143`） |

关键设计：`Backend.Execute` 见 `agent.go:17`；`New(agentType, Config)` 工厂见 `agent.go:297`，是一个 18 分支的 switch——每个分支返回独立的 `claudeBackend` / `codexBackend` / `qoderBackend` 等。

## 二、三种启动协议

18 个 provider 按与 CLI 的交互协议可分为三大类。协议决定了 Multica 能拿到什么粒度的数据（流式事件 vs 仅最终输出）、如何注入上下文（磁盘文件 vs 内联参数）、如何处理取消与恢复。

| 协议 | 原理 | Provider | 优点 | 局限 |
|------|------|----------|------|------|
| **ACP**（Agent Client Protocol） | 通过 JSON-RPC 的持久化协议与 CLI 通信，服务端推送工具调用、客户端流式返回 | kimi、kiro、hermes、qoder、qoderclicn、traecli、grok、openclaw、copilot | 标准化、结构化事件、可做超时控制与进度追踪 | 每个 CLI 的 ACP 实现成熟度不一，需 launch header 适配 |
| **Stream-JSON** | 解析 CLI 的 JSON 流（逐事件输出） | claude、codebuddy、cursor、opencode、qwen、pi | 事件粒度细、集成简单 | 依赖各 CLI 私有的输出格式 |
| **原生 / 专用** | 每个 CLI 专有的命令行模式 | codex（app-server RPC）、antigravity（`agy -p`）、deveco（`deveco run`） | 深度定制，能做充分隔离 | 强绑定特定 CLI 行为，维护成本高 |

> **实现说明**：以上分类基于 `launchHeaders` 映射（`server/pkg/agent/agent.go:353`）、各 backend 文件的实际调用方式（如 pi 走 `pi -p --mode json`、grok 走 `grok agent --always-approve stdio`）归纳，并非代码中的显式协议枚举。

## 三、完整 provider 清单

以下 18 个均来自 `agent.go` 的 `SupportedTypes` 权威白名单（`agent.go:237`）：

| Provider | 分类键 | 后端 CLI | 启动骨架（launch header） | 备注 |
|----------|--------|----------|---------------------------|------|
| `claude` | claude | `claude` | `claude (stream-json)` | 最老牌流式后端之一，模型/path/args 环境变量覆盖，hooks 自愈 |
| `codebuddy` | codebuddy | `codebuddy` | `codebuddy (stream-json)` | 字节跳动 CodeBuddy，写 `CODEBUDDY.md` |
| `codex` | codex | `codex` | `codex app-server` | **最大最复杂的后端**：sandbox、per-task HOME、记忆禁用、multi-agent 禁用、skill 剥离 |
| `copilot` | copilot | `copilot` | `copilot (json)` | GitHub Copilot CLI，ACP；Windows 调用特殊处理 |
| `opencode` | opencode | `opencode` | `opencode run (json)` | MCP 配置注入、空闲 watchdog、取消 |
| `deveco` | deveco | `deveco` | `deveco run (json)` | DevEco Code（华为），`deveco_models.go` 独立模型映射（migration 175） |
| `openclaw` | openclaw | `openclaw` | `openclaw agent (json)` | 有专用的 shim 与 gateway pin；支持 local/gateway 双模式（见 `agent.go:75`） |
| `hermes` | hermes | `hermes` | `hermes acp` | Nous Research Hermes，ACP；有 `hermes_home.go` 隔离 |
| `pi` | pi | `pi` | `pi (json mode)` | pi.dev CLI，走 `pi -p --mode json` 的 JSON 流 |
| `cursor` | cursor | `cursor-agent` | `cursor-agent (stream-json)` | Cursor Agent 无头模式，stdin unix/windows；`cursor_mcp.go` |
| `kimi` | kimi | `kimi` | `kimi acp` | Moonshot Kimi，ACP |
| `kiro` | kiro | `kiro-cli` | `kiro-cli acp` | ACP |
| `antigravity` | antigravity | `agy` | `agy -p (non-interactive)` | Google Agent Builder，conversation-id reader（见 `agent.go:290-291` 的不可检测列表） |
| `qoder` | qoder | `qodercli` | `qodercli --acp` | Qoder 国际版（migration 134） |
| `qoderclicn` | qoderclicn | `qoderclicn` | `qoderclicn --acp` | Qoder 中国版（migration 242），**与 qoder 共享同一 ACP 后端** |
| `traecli` | traecli | `traecli` | `traecli acp serve` | Trae（字节），独立后端（migration 136） |
| `grok` | grok | `grok` | `grok agent stdio` | xAI Grok Build，ACP `--always-approve stdio`（migration 179）；支持 reporting `CostUSDTicks` |
| `qwen` | qwen | `qwen` | `qwen -p (stream-json)` | 阿里 Qwen Code（migration 202），写 `QWEN.md`，支持 `--mcp-config` |

> **注意**：`migration 120` 的 `runtime_profile.protocol_family` CHECK 约束（`server/migrations/120_runtime_profile.up.sql:41`）还包含 `gemini`，但 `agent.go` 的 `SupportedTypes` 并无 `gemini` 后端——说明 `gemini` 出现在 CHECK 里但尚未有正式后端实现（本仓库无 `gemini.go`）。`agent.go` 的 18 项 `SupportedTypes` 才是权威运行时可执行清单。qoder 与 qoderclicn 是两个 provider 键但共享 `qoderBackend`（`agent.go:329`），它们让守护进程能独立检测和注册国际/中国版二进制。

## 四、关键工程边界

### 4.1 Resume-Rejection 可检测性白名单

不同 CLI 对"恢复会话被拒绝"的可发现程度不同。`agent.go:281` 定义了一个**主动加入（opt-in）**的区分表：

```go
var resumeRejectionUndetectable = map[string]bool{
    "antigravity": true,
    "copilot":     true,
    "cursor":      true,
    "deveco":      true,
    "opencode":    true,
}
```

设计要点：

- 这些后端的 `Result.ResumeRejected` 永远为 `false`，但语义是"无法判断"而非"未被拒绝"
- 不在表内的后端**默认视为可检测**——新后端 fail-closed：宁可不触发 fallback，也不靠猜测重试
- 表内后端从流输出里抠出 SessionID，但缺少结构化的拒绝信号（copilot 的 `session.error` 先于 `session.start` 到达时 SessionID 为空，`agent.go:273-274`）

### 4.2 跨运行时通用参数穿透

`ExecOptions` 定义了一批**只有部分后端消费、其余静默忽略**的参数（`agent.go:24-92`）：

| 参数 | 消费的后端 | 作用 |
|------|-----------|------|
| `ThinkingLevel` | claude、codex、opencode、codebuddy、grok | 运行时原生的推理强度（如 Claude 的 low↔max、Codex 的 none↔xhigh） |
| `ServiceTier` | codex | Codex 执行层级（如 priority/Fast） |
| `SystemPrompt` | 无法从磁盘读取 brief 的少数后端 | 内联系统提示；多数后端通过工作目录的上下文文件（CLAUDE.md/AGENTS.md/QWEN.md）交付，**新后端若只依赖它则静默收不到**（`agent.go:28-35`） |
| `McpConfig` | 支持 `--mcp-config` 的后端 | 注入 MCP server 配置 |
| `OpenclawMode` | openclaw | local（内嵌进程）或 gateway（routing）双模式 |

设计哲学：**"忽略而非失败"**——不打断无关 agent，让 thinking/service tier 等能力增量扩展到更多运行时（`agent.go:66-71` 的 MUL-2339 注释）。

### 4.3 超时与宕机看门狗

`ExecOptions` 携带多层超时（`agent.go`）：

- **`Timeout`**：绝对墙钟 deadline。`>0` 才生效；`0`/负数则无 deadline，交由看门狗兜底（MUL-3064，`agent.go:100-105`）——让持续产生事件的会话不会被"只因为它跑得久"而杀掉
- **`IdleWatchdogTimeout`**：窄化守护进程通用的"无消息"看门狗窗口；不能超过守护进程级安全上界
- **`SemanticInactivityTimeout`**：语义无进展超时（codex 专用，排除有毒上下文）
- **`HandshakeTimeout`**：限制长连接协议（codex app-server）的启动 RPC 时长

## 五、架构定位

将运行时抽象为 `pkg/agent` 是 8 月最大的演进之一：

1. **从"硬编码二选一"到"可枚举插件式"**：`New()` 工厂 + `SupportedTypes` 白名单，新增运行时 = 加一个 backend 文件 + 白名单 + migration（每个新 provider 都有对应 migration 扩展 CHECK 约束）
2. **与隔离层解耦**：未更新的 `daemon/execenv/`（进程隔离、上下文文件注入）与 `pkg/agent/`（执行协议）职责分离——前者负责"在一个干净环境里跑"，后者负责"如何与 CLI 说话"
3. **协议差异被统一**：无论 ACP 还是 stream-JSON，最终都归一为 `Message` 事件流 + `Result` 最终态，下游（WebSocket 推送、活动日志）无需感知 provider 差异

这也解释了 git log 里频繁出现的 runtime 相关的 commit（Qoder CN、kimi、kiro、traecli 等）——每次都是"新 backend + 白名单 + migration CHECK 扩展"的三件套。
