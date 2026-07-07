# OpenHuman

**AI 超级助手 — 面向 Web3 加密货币社区的桌面 Agent 平台**

| 元数据 | 值 |
|--------|-----|
| **语言** | Rust + TypeScript（React） |
| **框架** | Tauri v2 + Vite 7 + Redux Toolkit |
| **核心依赖** | tokio、axum、rusqlite、tinyagents（自研） |
| **协议** | AGPL-3.0 |
| **仓库** | [GitHub](https://github.com/tinyhumansai/openhuman) |
| **版本** | v0.58.12 |

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
| **状态管理** | Redux Toolkit + redux-persist | 41 个 slice，用户级 namespace 持久化 |
| **国际化** | 自研 i18n | 18 个语言环境，CI 强制翻译完备性 |
| **核心语言** | Rust（2021 edition） | 进程内嵌入，非 sidecar |
| **异步运行时** | tokio（full feature） | |
| **RPC 协议** | JSON-RPC 2.0（axum）+ Socket.IO（socketioxide） | 双通道通信 |
| **数据库** | SQLite（rusqlite bundled）+ PostgreSQL | |
| **图数据库** | Neo4j | 知识图谱实体关系 |
| **AI 框架** | tinyagents（自研） | LangGraph 风格 orchestration |
| **记忆引擎** | tinycortex（自研） | 向量 + FTS5 混合搜索 |
| **工作流引擎** | tinyflows（自研） | 节点图工作流 |
| **语音** | whisper-rs（metal 加速） | STT + TTS |
| **加密** | AES-256-GCM / Argon2id / XChaCha20-Poly1305 / X25519 | 全链路加密 |
| **钱包** | ethers（EVM）/ bitcoin / solana（ed25519-dalek） | 多链支持 |
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
│  │  → 135 domain modules                  │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

### 3.2 135 域模块系统

遵循 Unix 风格的单职责模块设计，每个域的标准布局：

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

事件域：Agent、Memory、Channel、Cron、Skill、Tool、Webhook、System

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

| 参数 | 值 |
|------|-----|
| 分块 | 512 tokens/chunk，64-token overlap |
| 嵌入 | OpenAI `text-embedding-3-small` |
| 搜索混合比例 | 70% 向量相似度 + 30% FTS5 全文 |
| 知识图谱 | Neo4j 实体关系 |

### 3.6 Agent 框架（tinyagents）

三种 Agent 原型：

- **Orchestrator**：多任务管理，分解+调度+综合
- **Specialist**：专用 Worker，一次性驱动
- **Reactor**：事件驱动，由触发器唤醒

触发流程：小型本地 LLM 分类 → drop / notify / spawn_reactor / spawn_orchestrator

### 3.7 iOS 客户端（实验性）

不包含本地 Rust 核心，纯传输层客户端。三种连接策略：

1. **LanHttpTransport**：局域网直连
2. **TunnelTransport**：E2E 加密隧道（XChaCha20-Poly1305）
3. **CloudHttpTransport**：云端中继

配对流程：QR 码扫描 → X25519 密钥协商 → 对称加密通信

---

## 四、关键数据

| 维度 | 数据 |
|------|------|
| **域模块数** | 135 |
| **前端 slice** | 41 |
| **前端路由** | 20+（HashRouter） |
| **Rust 二进制** | 7 个（含测试二进制） |
| **vendored 子模块** | 6 个（tinyagents、tinyflows、tinycortex、tinyjuice、tinychannels、tinyplace） |
| **测试文件** | 132 个 Rust E2E 测试 |
| **CI 通道** | 双通道（CI Lite / CI Full） |
| **覆盖率门禁** | ≥ 80% 变更行覆盖率 |
| **国际化语言** | 18 个 |
| **GitBook 文档** | 15+ 篇 |

---

## 五、技术债务与风险

| 问题 | 影响 | 状态 |
|------|------|------|
| **双 Cargo 世界** | 构建复杂度高，两个 target/ 目录 | [PR #3877](https://github.com/tinyhumansai/openhuman/pull/3877) 计划收敛 |
| **vendored CEF CLI** | 标准 `@tauri-apps/cli` 不可用，维护成本高 | 长期持有 |
| **macOS Apple Silicon 兼容** | whisper-rs / llama.cpp 需 `GGML_NATIVE=OFF` | workaround |
| **135 域边界** | 部分域（agent/）体量过大，边界模糊 | 持续演进 |
| **iOS 客户端实验性** | 传输协议需随核心演进同步维护 | 实验阶段 |
| **自研框架过多** | tinyagents/tinycortex/tinyflows 等自研组件增加学习和调试成本 | 战略选择，垂直深度换取控制力 |

---

## 六、总结

OpenHuman 是一个**技术密度极高**的桌面 Agent 平台。其核心特征：

1. **技术深度**：自研 AI 编排框架、记忆引擎、工作流引擎、token 压缩引擎，非简单包装 LLM API
2. **隐私优先**：本地核心 + 本地存储 + OS 密钥环，敏感数据不离用户机器
3. **Web3 原生**：内置钱包（EVM/Bitcoin/Solana）、链上交互（swap/bridge/dapp）
4. **架构复杂度高**：135 域模块 + 双 Cargo 世界 + vendored 子模块，是学习 Rust 大型项目架构的优秀案例
5. **安全隐患意识**：多层级安全模型（权限分级 + 审批门控 + 沙箱 + 路径隔离 + 提示注入防护）

**竞品对比定位**：偏向重度用户 / 加密货币社区的桌面 Agent（类似 ReWorkd AgentGPT 的定位，但更强调本地优先和 Web3 集成），而非服务企业团队的 SaaS Agent。
