# OpenHuman

> **本版增量更新说明**：本文基于当前工作区 **v0.63.6**（上游 `git describe` 指向 v0.63.1-187；HEAD `e29bfc66f`），较初版 v0.58.12 有重大演进。本次依据代码快照比对（非官方 changelog）同步了下列数据：版本号；域模块从 135 扁平域收敛为 **22 个 `DomainGroup` 族（85 个注册模块）**（上游提交 `b668251c5` collapse 124 flat domains）；前端 Redux slice 从 41 → 21；**移除已不使用的 Neo4j / PostgreSQL**；vendored 子模块从 6 → 9；Rust 二进制 7 → 10（含 feature 门控）；补充了子conscious/flows/inference 等新模块。原「三、架构特色」各小节结构保留，新增内容以 `> 演进说明` 标注。

**AI 超级助手 — 面向 Web3 加密货币社区的桌面 Agent 平台**

| 元数据 | 值 |
|--------|-----|
| **语言** | Rust + TypeScript（React） |
| **框架** | Tauri v2 + Vite 7 + Redux Toolkit |
| **核心依赖** | tokio、axum、rusqlite、tinyagents / tinycortex / tinyflows（自研，vendored） |
| **协议** | AGPL-3.0 |
| **仓库** | [GitHub](https://github.com/tinyhumansai/openhuman) |
| **版本** | v0.63.6（上游最新 tag v0.63.9） |

---

## 子文档索引

- [`memory-system.md`](memory-system.md) — 记忆系统深度分析（已更新至 v0.63 架构演进）
- [`orchestration.md`](orchestration.md) — Agent 编排层：triage 四分类、harness checkpointed graph、subagent fleet
- [`model-routing.md`](model-routing.md) — 推理/模型路由：workload→tier→provider 三层路由与 per-call cost
- [`flows.md`](flows.md) — 工作流引擎：approval-gated、trigger-driven、durable 自动化
- [`subconscious.md`](subconscious.md) — 潜意识子系统：cron 驱动的 Deep Reflection 与双 world 实现

---

## 一、项目定位

OpenHuman 是一个面向加密货币社区的 **AI 超级助手桌面应用**。用户通过桌面运行一个本地 Rust 核心引擎，AI Agent 可以接入用户的 Telegram、Discord、Slack、Gmail、WhatsApp 等渠道，提供会话式 AI 助手 + 自动化操作 + 链上交互能力。

它不是 SaaS 云服务，而是**本地优先**的用户代理软件 — 核心逻辑在用户机器上运行，敏感凭证存储在 OS 密钥环中，通过 Tauri 桌面外壳封装。

---

## 二、技术栈

| 层 | 技术选型 | 备注 |
|----|----------|------|
| **桌面框架** | Tauri v2（CEF webview） | 使用 vendored CLI，标准 `@tauri-apps/cli` 构建会损坏 |
| **前端** | React 19 + TypeScript 5.8 | Vite 7 构建 |
| **样式** | Tailwind CSS | 主题 token：ocean `#4A83DD`，Inter + Cabinet Grotesk 字体 |
| **状态管理** | Redux Toolkit（+ redux-persist） | 21 个 createSlice，用户级 namespace 持久化；未用 RTK Query |
| **国际化** | 自研 i18n | 18 个语言环境，CI 强制翻译完备性 |
| **核心语言** | Rust（2021 edition） | 进程内嵌入，非 sidecar |
| **异步运行时** | tokio（full feature） | |
| **RPC 协议** | JSON-RPC 2.0（axum）+ Socket.IO（socketioxide） | 双通道通信 |
| **数据库** | SQLite（rusqlite bundled） | 本地核心主要后端 |
| **知识图谱** | 记忆树 + SQLite FTS（Neo4j 已不再使用） | 初版所称 Neo4j 当前代码 0 处命中 |
| **AI 编排框架** | tinyagents（vendored crate） | LangGraph 风格 checkpointed graph orchestration |
| **记忆引擎** | tinycortex（vendored crate） | host 侧 `memory/` 为编排/策略/adapter 层 |
| **工作流引擎** | tinyflows（vendored crate） | 节点图工作流，via `flows/` 编排 |
| **语音** | whisper.cpp（whisper-rs metal 加速）+ Piper TTS | STT + TTS |
| **加密** | AES-256-GCM / Argon2id / XChaCha20-Poly1305 / X25519 | 全链路加密 |
| **钱包** | ethers（EVM）/ bitcoin / solana（ed25519-dalek）+ x402 | 多链支持 |
| **监控** | Sentry + OpenTelemetry + Prometheus | |
| **包管理** | pnpm monorepo + cargo（双 Cargo 世界） | 两个独立 target/ |

---

## 三、架构特色

### 3.1 进程内嵌入式核心

核心不以独立进程（sidecar）运行，而是作为 Tauri 应用内的一个 tokio 任务。通过网络端口暴露 HTTP JSON-RPC 端点供前端访问，`CoreProcessHandle` 管理生命周期。Token 通过内存传递而非文件系统。

```
┌─────────────────────────────────────────────┐
│  Tauri Desktop App                          │
│  ┌───────────────────────────────────────┐  │
│  │  React Frontend (HashRouter SPA)      │  │
│  │  core_rpc_relay invoke →              │  │
│  └──────────┬────────────────────────────┘  │
│             │ HTTP 127.0.0.1:<port>/rpc     │
│  ┌──────────▼────────────────────────────┐  │
│  │  Rust Core (tokio task)               │  │
│  │  axum JSON-RPC server                │  │
│  │  → Controller dispatch                 │  │
│  │  → 22 DomainGroup 族 / 85 域模块       │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

### 3.2 域模块系统（DomainGroup）

> **演进说明(v0.63)**：初版所称"135 个扁平域"已演化。上游提交 `b668251c5`（2026-08-03）执行 "collapse flat domains into gate-aligned families"，现收敛为 **22 个 `DomainGroup` 族**，覆盖 **85 个注册域模块**（`src/core/all.rs:148` `DomainGroup::COUNT = 22`；注册数组 `all.rs:159-182`）。每个族对齐业务目录（`agent/`、`memory/`、`inference/`、`flows/`、`subconscious/` 等）。单个域的标准布局仍为：

| 文件 | 职责 |
|------|------|
| `mod.rs` | 仅导出 + controller schema 对 |
| `types.rs` | Serde 类型定义 |
| `store.rs` | 持久化层 |
| `ops.rs` | 业务逻辑 + `RpcOutcome<T>` |
| `schemas.rs` | RPC 控制器 + `handle_*` |
| `tools.rs` | Agent 工具实现 |
| `bus.rs` | 事件总线处理器 |

所有域通过 `ControllerSchema` 注册到 `src/core/all.rs`，自动分配到 JSON-RPC 和 CLI。

### 3.3 事件总线

类型化发布/订阅系统，支持两种模式：

- **广播（publish/subscribe）**：fire-and-forget，多订阅者
- **原生请求/响应（native request/response）**：一对一，零序列化，内部域间通信

> **演进说明(v0.63)**：事件域已远超初版的 8 个，`src/core/event_bus/events.rs` 现有约 31 个域标记段——除 Agent、Memory、Channel、Cron、Skill、Tool、Webhook、System 外，新增 Subconscious、Run Queue、Monitor、Memory Diff、Tree Summarizer、Voice、Flows、Inference 等。broadcast / native req-resp 两种模式机制不变。

事件域（概览）：Agent、Subconscious、Memory、Memory Diff、Channels、Cron、Skills、Tools、Webhooks、Flows、System 等 31 个域段

### 3.4 安全体系

| 层次 | 机制 |
|------|------|
| **权限模型** | read-only / supervised / full 三档 + workspace_only / trusted_roots |
| **审批门控** | 默认开启，后台任务自动放行，10 分钟 TTL 自动拒绝 |
| **命令分类** | Read / Write / Network / Install / Destructive |
| **沙箱** | Docker / Landlock（Linux）/ Bubblewrap / Noop |
| **凭证存储** | OS 密钥环（macOS Keychain、Windows Credential Manager、Linux Secret Service） |
| **路径隔离** | 工作区目录 `~/.openhuman/users/<id>/workspace` 不可写入 |

### 3.5 记忆系统（tinycortex）

> **演进说明(v0.63)**：记忆系统已从 host 自研模块迁移至 vendored crate **tinycortex**，host 侧仅存编排/策略/adapter 层（详见 [`memory-system.md`](memory-system.md)）。下列参数为历史快照参考；实际引擎本尊在 `vendor/tinycortex`，embedding 默认 `embedding-v1`（voyage-3.5）/ Ollama `bge-m3`。

| 参数 | 值 |
|------|-----|
| 分块 | 512 tokens/chunk，64-token overlap |
| 嵌入 | OpenAI `text-embedding-3-small`（历史）；当前 `embedding-v1` / Ollama `bge-m3` |
| 搜索混合比例 | 70% 向量相似度 + 30% FTS5 全文 |
| 知识图谱 | 记忆树 + SQLite FTS（初版所称 Neo4j 已不再使用） |

### 3.6 Agent 框架（tinyagents + harness）

> **演进说明(v0.63)**：Agent 执行现跑在发布版 `tinyagents` crate 之上（checkpointed `CompiledGraph`，`SqliteCheckpointer` 断点续跑），host 侧 `agent/` 是 adapter seam + 编排控制面（详见 [`orchestration.md`](orchestration.md)）。触发流程的"三分类"已演进为 **四分类** `Drop` / `Acknowledge` / `React` / `Escalate`（`src/openhuman/agent/triage/decision.rs`）。Agent 原型（Orchestrator / Specialist / Reactor）仍是三种，但编排走 durable graph。

三种 Agent 原型：

- **Orchestrator**：多任务管理，分解+调度+综合（深推理核心，委派 worker fleet）
- **Specialist**：专用 Worker，一次性驱动
- **Reactor**：事件驱动，由触发器唤醒

触发流程：小型本地 LLM 分类（triage）→ **Drop（丢弃） / Acknowledge（记录，不派 agent） / React（单一副作用） / Escalate（多步委派给 orchestrator）**

### 3.7 iOS 客户端（实验性）

不包含本地 Rust 核心，纯传输层客户端。三种连接策略：

1. **LanHttpTransport**：局域网直连
2. **TunnelTransport**：E2E 加密隧道（XChaCha20-Poly1305）
3. **CloudHttpTransport**：云端中继

配对流程：QR 码扫描 → X25519 密钥协商 → 对称加密通信

---

## 四、关键数据

| 维度 | 数据（v0.63 核对） |
|------|------|
| **域族 / 模块** | 22 个 `DomainGroup` 族 / 85 个注册域模块 |
| **前端 slice** | 21 个 createSlice（未用 RTK Query） |
| **前端路由** | 20+（HashRouter） |
| **Rust 二进制** | 10 个 `[[bin]]`（多数 feature 门控） |
| **vendored 子模块** | 9 个（tauri-cef、tauri-plugin-notification、tinyagents、tinyflows、tinycortex、tinyjuice、tinychannels、tinyplace、tinyhumans-sdk） |
| **顶层模块** | 31 个（agent / memory / subconscious / flows / inference / channels / voice / web3 / threads / media …） |
| **事件域段** | 约 31 个 |
| **测试文件** | 大规模 Rust E2E（含记忆 golden parity） |
| **CI 通道** | 双通道（CI Lite / CI Full） |
| **覆盖率门禁** | ≥ 80% 变更行覆盖率 |
| **国际化语言** | 18 个 |
| **GitBook 文档** | 15+ 篇 |

---

## 五、技术债务与风险

| 问题 | 影响 | 状态 |
|------|------|------|
| **双 Cargo 世界** | 构建复杂度高，两个 target/ 目录 + vendored submodule 依赖 | [PR #3877](https://github.com/tinyhumansai/openhuman/pull/3877) 计划收敛 |
| **vendored CEF CLI** | 标准 `@tauri-apps/cli` 不可用，维护成本高 | 长期持有 |
| **macOS Apple Silicon 兼容** | whisper.cpp / llama.cpp 需 `GGML_NATIVE=OFF` | workaround |
| **域族边界** | 初版 135 扁平域已收敛为 22 族，但部分族（agent/）体量仍大，边界持续演进 | 持续演进 |
| **iOS 客户端实验性** | 传输协议需随核心演进同步维护 | 实验阶段 |
| **自研框架过多** | tinyagents/tinycortex/tinyflows 等自研组件（多为 vendored submodule）增加学习和调试成本 | 战略选择，垂直深度换取控制力 |

---

## 六、总结

OpenHuman 是一个**技术密度极高**的桌面 Agent 平台。其核心特征：

1. **技术深度**：自研 AI 编排框架、记忆引擎、工作流引擎、token 压缩引擎，非简单包装 LLM API
2. **隐私优先**：本地核心 + 本地存储 + OS 密钥环，敏感数据不离用户机器
3. **Web3 原生**：内置钱包（EVM/Bitcoin/Solana）、链上交互（swap/bridge/dapp）
4. **架构复杂度高**：22 域族/85 模块 + 双 Cargo 世界 + vendored 子模块，是学习 Rust 大型项目架构的优秀案例
5. **安全隐患意识**：多层级安全模型（权限分级 + 审批门控 + 沙箱 + 路径隔离 + 提示注入防护）

**竞品对比定位**：偏向重度用户 / 加密货币社区的桌面 Agent（类似 ReWorkd AgentGPT 的定位，但更强调本地优先和 Web3 集成），而非服务企业团队的 SaaS Agent。
