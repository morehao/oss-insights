# Agent 产品

本目录收录 Agent 产品（Agent 协作平台、AI 原生团队工具）领域的开源项目解读。

| 项目 | 语言 | 说明 | 协议 | 仓库 |
|------|------|------|------|------|
| [OpenWorker](openworker/) | Python+TS+Rust | 开源 AI 同事桌面平台，本地 sidecar 架构，审批门控 Agent 循环，5+ LLM Provider（+Together/Fireworks 转售），26+ 连接器，MCP 协议，技能 CRUD/上传/自定义，上下文自动压缩，SSRF 防护 | MIT | [GitHub](https://github.com/andrewyng/openworker) |
| [OpenHuman](openhuman/) | Rust + TS | Web3 社区 AI 超级助手桌面应用，进程内 Rust 核心（135个域模块），自研 AI 编排/记忆/工作流引擎，多链钱包，MCP 协议 | AGPL-3.0 | [GitHub](https://github.com/tinyhumansai/openhuman) |
| [PicoClaw](picoclaw/) | Go | 极致轻量 Go 原生 AI Agent 网关，<10MB 内存跑在 $10 硬件，20+ 通道 30+ LLM，MCP 集成 | MIT | [GitHub](https://github.com/sipeed/picoclaw) |
| [Leros](leros/) | Go | 企业级 AI 工作协作平台，四平面架构（Edge/Interface/Control/Execution），支持多 Agent 运行时（Eino/Claude Code/Codex CLI），事件驱动 + at-least-once 崩溃恢复。本解读基于私有 GitLab fork（corekg/lework，v0.3.7） | Apache-2.0 | [GitHub](https://github.com/insmtx/Lework) |
| [Multica](multica/) | Go+TS | 托管式 AI Agent 协作平台，Agent 作为第一公民的任务管理系统 | Apache-2.0 | [GitHub](https://github.com/multica-ai/multica) |
| [OpenBee](openbee/) | Go+TS | 自托管 24/7 数字员工管理平台，对话驱动，多 IM 平台集成，Worker 具有组织身份 | MIT | [GitHub](https://github.com/theopenbee/openbee) |
| [OpenVort](openvort/) | Python+TS | 开源 AI 员工平台/研发工作流引擎，插件化架构，深度绑定 IM + 项目管理/代码/CI/CD 全流程 | AGPL-3.0 | [GitHub](https://github.com/openvort/openvort) |
| [OpenCode](opencode/) | TypeScript | 开源 AI 编码 Agent，服务化架构（CLI/TUI/Web/Desktop），四轴可组合 LLM 路由，事件溯源 Session V2，Effect-TS 函数式编排 | MIT | [GitHub](https://github.com/anomalyco/opencode) |
| [DeepWiki-Open](deepwiki-open/) | Python+TS | 开源代码仓库 Wiki 自动生成平台，Next.js+FastAPI+AdalFlow，9 种 LLM Provider，RAG+Deep Research+Codemap 引用落点 | MIT | [GitHub](https://github.com/AsyncFuncAI/deepwiki-open) |
| [Mem0](mem0/) | Python+TS | AI Agent 智能记忆层，70+ Provider 插件体系，V3 多信号检索，双模式部署 + MCP 集成 | Apache-2.0 | [GitHub](https://github.com/mem0ai/mem0) |
| [Memos](memos/) | Go+TS | 自托管笔记应用，微博式时间线，Proto 驱动 API，双重协议（Connect RPC + gRPC-Gateway），三数据库支持，SSE 实时推送，Go+React 工程实践范本 | MIT | [GitHub](https://github.com/usememos/memos) |
| [OpenWork](openwork/) | TS + Electron | 开源 AI 工作流分享桌面应用，Claude Cowork/Codex 开源替代；代理运行时+单一 baseUrl 透明代理，Den 企业控制平面，Drizzle+MySQL 约 80 表，MCP 两工具契约（search/execute），多云 Helm 部署 + 证据卷轴 evals | Fair Source（主体）/ ee 目录 Fair Source | [GitHub](https://github.com/different-ai/openwork) |
