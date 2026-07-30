# OpenWorker — 开源 AI 同事（AI Coworker）桌面平台深度解读

> **GitHub:** [github.com/andrewyng/openworker](https://github.com/andrewyng/openworker)
>
> **作者:** Andrew Ng（吴恩达）团队
>
> **关联文档:** [核心引擎与 Agent 体系](./engine-and-agent.md) · [连接器与 LLM 提供商体系](./connectors-and-providers.md) · [权限与安全模型](./permission-and-safety.md)

---

## 一、项目概览

### 这是什么

OpenWorker 是一款**开源的 AI 同事（AI Coworker）**桌面应用。与 ChatGPT、Claude 等 AI 助手不同，它的定位是**替代一个实习生或初级同事**——能跨桌面文件、终端命令行、以及 25+ 第三方服务（Slack/GitHub/Jira/Notion 等）独立开展工作，最终交付可用的产出物（文档、报表、代码、网页），而非仅给出建议或待办清单。

核心价值主张：**给 AI 一个任务，它执行，你验收结果。**

### 解决的痛点

现有 AI 工具多停留在"对话建议"阶段——用户需要自己记笔记、执行操作、在多个工具间复制粘贴。OpenWorker 要解决的是：

1. **跨工具协同**：AI 能同时操作文件系统、Shell 终端、Slack 消息、GitHub PR、Notion 文档
2. **交付成品**：AI 能执行完整的工作流程并产出可交付物，而非仅提建议
3. **自主工作模式**：支持无人值守（unattended）和定时调度，AI 能主动工作而不需要逐条指令
4. **安全边界**：通过细粒度权限和审批门控，让 AI 在安全范围内自主执行

### 核心能力清单

| 能力 | 说明 |
|------|------|
| **多 Agent 角色** | Code（编程）/ Cowork（知识工作）/ Chat（对话）/ MyHelper（个人助手）四种 Agent |
| **多 LLM 提供商** | OpenAI / Anthropic / Gemini / Vertex AI / Bedrock，统一路由层 |
| **25+ 连接器** | Slack / GitHub / Jira / Notion / Gmail / Google Calendar / Outlook / Linear 等 |
| **审批门控** | 4 种权限模式：Discuss / Plan / Interactive / Auto + 自定义 |
| **MCP 协议** | 完全支持 Model Context Protocol（stdio + streamable HTTP） |
| **技能系统** | 可加载的能力包（Anthropic 格式），渐进式暴露 |
| **持久记忆** | SQLite 存储，全局 + 工作区双作用域 |
| **定时任务** | cron 驱动的自动化调度（早报/周报/监控） |
| **本地优先** | 所有数据本地存储，仅模型调用和集成传出数据 |
| **语音输入** | Rust STT 边车进程支持语音转文字 |

### 技术栈一览

| 层面 | 技术 |
|------|------|
| 后端运行时 | Python 3.10+, FastAPI + Uvicorn |
| AI 框架 | aisuite（chat-completions + agents 框架） |
| 前端 | React 18, TypeScript, Vite 5, Tailwind CSS |
| 桌面壳层 | Tauri 2（macOS DMG / Windows 安装包） |
| 语音边车 | Rust（Cargo, `stt/` 目录） |
| 测试 | Vitest, Playwright, E2E |
| 打包 | PyInstaller（Python 后端打包） |
| 认证 | Auth0 OAuth（云服务） |
| CI/CD | GitHub Actions |

---

## 二、同类对比

| 项目 | 协议 | 定位 | 桌面端 | 工具集成 | LLM Provider | 审批控制 | MCP 支持 |
|------|------|------|--------|----------|-------------|---------|---------|
| **OpenWorker** | MIT | AI 同事平台 | ✅ Tauri 桌面 | 25+ 连接器 | 5+ | ✅ 4 级 | ✅ |
| **Claude Code** | 专有 | AI 编码 Agent | ❌ CLI only | 有限 | ❌ 独占 | ✅ | ❌ |
| **OpenCode** | MIT | AI 编码 Agent | ✅ Electron | 30+ 内置工具 | 20+ | ✅ 内置 | ✅ |
| **Cline** | Apache-2.0 | IDE 编码 Agent | ❌ VS Code | MCP 工具 | ✅ | ✅ | ✅ |
| **Leros** | Apache-2.0 | 企业编排平台 | ❌ Web | 10+ 连接器 | ✅ 多运行时 | ✅ | ✅ |
| **OpenVort** | AGPL-3.0 | 研发工作流引擎 | ❌ Web | IM + 代码 + CI/CD | ✅ | ❌ | ❌ |

**核心差异：** OpenWorker 是唯一以"桌面端 AI 同事"为定位的——它不只是编码 Agent（如 Claude Code/OpenCode），也不只是编排平台（如 Leros），而是一个安装在你电脑上、能操作你所有工具、像同事一样独立工作的 AI 应用。

---

## 三、项目规模

| 类别 | 数量 |
|------|------|
| Python 模块 | ~70+ 文件 |
| 测试文件 | ~84 个 |
| 前端包 | React + Tauri |
| 连接器数 | 25+ |
| LLM Provider | 5+ |
| GitHub Stars | ~1,500+（公开 Beta） |
| 许可证 | MIT |

---

## 四、架构总览

OpenWorker 采用**本地 sidecar 架构**：Python 后端作为本地 HTTP/WS 服务器运行（127.0.0.1），React 桌面 GUI 通过 WebSocket 通信，形成闭环。

```
┌─────────────────────────────────────────────────┐
│                 桌面 GUI (Tauri)                   │
│  React 18 + TailwindCSS                          │
│  ┌────────────────┐  ┌──────────────────────┐   │
│  │    聊天面板      │  │  审批面板/工具面板    │   │
│  │  WebSocket ←→  │  │  文件预览/连接器管理   │   │
│  └───────┬────────┘  └──────────────────────┘   │
└──────────┼──────────────────────────────────────┘
           │ WebSocket / HTTP
           ▼
┌─────────────────────────────────────────────────────┐
│              Python 后端 (FastAPI Sidecar)            │
│                                                       │
│  ┌───────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ Session   │  │ Permission│  │  Event Bus       │  │
│  │ Manager   │  │ Engine   │  │  (引擎事件流)     │  │
│  └─────┬─────┘  └──────────┘  └──────────────────┘  │
│        │                                              │
│  ┌─────▼────────────────────────────────────────┐    │
│  │           TurnEngine (Agent 循环)              │    │
│  │  用户输入 → LLM 推理 → 工具调用 → 继续 → 完成 │    │
│  └─────┬────────────────────────────────────────┘    │
│        │                                              │
│  ┌─────┴──────────┬──────────┬──────────────────┐    │
│  │  Tool Registry  │  Connector│  MCP Client      │    │
│  │  (10+ 内置工具)  │  Manager  │  (stdio/http)   │    │
│  └────────────────┘  (25+ 连接器)└──────────────────┘  │
│        │                                              │
│  ┌─────┴──────────────────┐                           │
│  │  Provider Router        │                           │
│  │  OpenAI/Anthropic/...   │                           │
│  └─────────────────────────┘                           │
└─────────────────────────────────────────────────────────┘
           │
           ▼
   ┌───────────────┐  ┌──────────────────┐
   │ 本地文件系统    │  │ 第三方服务 (25+)  │
   │ Shell 终端      │  │ Slack/GitHub/... │
   └───────────────┘  └──────────────────┘
```

### 核心执行链路

```
用户通过 GUI/TUI/Slack/Telegram 发起请求
           │
           ▼
    ┌───────────────────────────┐
    │   FastAPI WebSocket/REST  │
    └───────────┬───────────────┘
                │
                ▼
    ┌───────────────────────────┐
    │    SessionManager         │  ← 创建/复用会话
    │    ┌─────────────────┐    │
    │    │  build_engine() │    │  ← 装配所有组件
    │    └────────┬────────┘    │
    └─────────────┼─────────────┘
                  │
                  ▼
    ┌────────────────────────────────────────────┐
    │            TurnEngine (Agent 循环)          │
    │                                            │
    │  1. context_provider()  ← 注入上下文       │
    │  2. provider.chat()     ← LLM 推理         │
    │  3. 检查模型输出:                           │
    │     ├─ 文本回复 → 发送给 surface           │
    │     └─ 工具调用 → PermissionEngine 判断     │
    │          ├─ ALLOW  → 并发/顺序执行工具       │
    │          ├─ NEEDS_USER → 发出审批事件       │
    │          └─ DENY  → 拒绝并通知              │
    │  4. 工具结果 → 继续第2步（最多 max_iter）    │
    │  5. 完成 → 发送最终回复 + 事件              │
    └────────────────────────────────────────────┘
```

### 包间依赖

```
coworker/                  ← Python 后端主包（~70+ 文件）
├── agents/                ← Agent 定义（Code/Cowork/Chat/MyHelper）
├── providers/             ← LLM Provider 适配层
├── connectors/            ← 25+ 第三方集成连接器
├── tools/                 ← 内置工具集（文件/Shell/Git/搜索等）
├── mcp/                   ← MCP 协议客户端
├── memory/                ← 持久记忆系统
├── skills/                ← 技能加载系统
├── personas/              ← 角色（Persona）系统
├── automation/            ← 定时任务调度
├── server/                ← FastAPI 服务器 + Session Manager
├── web/                   ← Web 工具（Fetch/搜索）
└── tui/                   ← 终端 UI

surfaces/gui/              ← React/Tauri 桌面前端
stt/                       ← Rust 语音转文字边车
packaging/                 ← 安装包打包脚本
tests/                     ← 测试（84+ 文件）
```

---

## 五、设计亮点

### 1. 审批门控式 Agent 循环

核心创新在 PermissionEngine 与 TurnEngine 的协作：不是简单的"全部放行"或"全部审批"，而是根据风险等级（read/write/shell/send）动态决策。低风险操作（读文件、搜索）自动放行且有并发优化；高风险操作（写文件、执行命令）等待用户审批。这是目前开源 AI Agent 中最完备的权限模型之一。

### 2. 连接器作为插件一等公民

25+ 连接器不是硬编码集成，而是通过统一的 Adapter 模式、Gateway 消息分发、以及 **relay_client** WebSocket 中继实现。每个连接器可独立配置、独立认证、独立路由。这种设计使得新增一个连接器不需要修改核心引擎。

### 3. Provider Router + 统一抽象

通过 `ProviderClient` 抽象基类 + `ProviderRouter` 按模型字符串前缀路由（如 `openai/`、`claude/`、`ollama/`），实现了厂商无关的 LLM 调用。这是 Andrew Ng 团队另一项目 `aisuite` 的自然延伸。

### 4. 本地优先 + 云可选

所有数据（对话历史、密钥配置、连接器 Token）默认存储在本地文件系统。云服务（Auth0 认证、连接器中继）是可选项。这种设计既满足了企业数据安全要求，又保留了云服务的便利性。

---

## 六、不足

- **Python 性能瓶颈**：核心 agent 循环是同步阻塞调用（通过 `asyncio.to_thread` 包装），高并发场景下受限
- **桌面端绑定**：目前紧密绑定 Tauri 桌面，CLI/TUI 功能有限
- **连接器认证复杂**：每个连接器的 OAuth 流程独立管理，部署和配置有一定复杂性
- **快速迭代中**：项目处于公开 Beta 阶段，API 和模块结构可能变化
- **文档待完善**：优秀的设计缺乏体系化的技术文档

---

## 七、可借鉴的设计

1. **审批门控 + 风险分级**：将 AI 的每个操作按风险分类，对不同风险级别采用不同的审批策略——这是构建可信 AI Agent 的工程范式
2. **连接器 Adapter 模式**：通过 Adapter + Gateway 解耦消息分发与具体实现，适合需要集成多个外部服务的系统
3. **分层配置**：全局配置 < 工作区配置，天然支持多项目/多工作区场景
4. **本地优先架构**：关键数据默认存本地，云作为可选增强而非必需依赖，符合企业级部署要求

---

> **关联文档:** [核心引擎与 Agent 体系](./engine-and-agent.md) · [连接器与 LLM 提供商体系](./connectors-and-providers.md) · [权限与安全模型](./permission-and-safety.md)
