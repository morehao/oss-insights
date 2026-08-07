# Agent 产品

本目录收录 Agent 产品（Agent 协作平台、AI 原生团队工具）领域的开源项目解读。

| 项目 | 语言 | 说明 | 协议 | 仓库 |
|------|------|------|------|------|
| [OpenWorker](openworker/) | Python+TS+Rust | 开源 AI 同事桌面平台，本地 sidecar 架构，审批门控 Agent 循环，5+ LLM Provider（+Together/Fireworks 转售），26+ 连接器，MCP 协议，技能 CRUD/上传/自定义，上下文自动压缩，SSRF 防护 | MIT | [GitHub](https://github.com/andrewyng/openworker) |
| [OpenHuman](openhuman/) | Rust + TS | 本地优先 AI 超级助手桌面应用：进程内 Rust 核心（22 域族/85 模块），分脑编排（triage 四分类 + checkpointed tinyagents graph），记忆树+Obsidian/TinyCortex 记忆引擎，per-call 成本与模型路由，approval-gated 工作流，多链钱包，MCP 协议 | AGPL-3.0 | [GitHub](https://github.com/tinyhumansai/openhuman) |
| [PicoClaw](picoclaw/) | Go | 极致轻量 Go 原生 AI Agent 网关，<10MB 内存跑在 $10 硬件，20+ 通道 30+ LLM，MCP 集成 | MIT | [GitHub](https://github.com/sipeed/picoclaw) |
| [Lework](Lework/) | Go+TS+Electron | 企业级 AI 工作协作平台（AI 劳动力管理），双进程事件驱动（Server/Worker + NATS at-least-once），多 Agent 运行时（native/Claude Code/Codex/OpenCode），Task+Asset 模型，Skill/Tool + MCP/插件，企业级权限/审计。本解读基于私有 GitLab fork（corekg/lework/Lework-fork） | Apache-2.0 | [GitHub](https://github.com/insmtx/Lework) |
| [Multica](multica/) | Go+TS | 托管式 AI Agent 协作平台，Agent 作为第一公民的任务管理系统 | Apache-2.0 | [GitHub](https://github.com/multica-ai/multica) |
| [OpenBee](openbee/) | Go+TS | 自托管 24/7 数字员工管理平台，对话驱动，多 IM 平台集成，动态引擎路由（Claude/Codex/Pi 运行时切换），多用户 RBAC，49 条 SQLite 迁移，多渠道分发 | MIT | [GitHub](https://github.com/theopenbee/openbee) |
| [OpenVort](openvort/) | Python+TS | 开源 AI 员工平台/研发工作流引擎，手写 Agentic Loop（空操作检测/多模型 failover），插件化架构 + Slot Protocol，四级 Skill 体系，无 Alembic 自动迁移，MCP Server（tools 维度），深度绑定 IM + 项目管理/代码/CI/CD 全流程 | AGPL-3.0 | [GitHub](https://github.com/openvort/openvort) |
| [OpenCode](opencode/) | TypeScript | 开源 AI 编码 Agent，服务化架构（CLI/TUI/Web/Desktop），四轴可组合 LLM 路由，事件溯源 Session V2，Effect-TS 函数式编排 | MIT | [GitHub](https://github.com/anomalyco/opencode) |
| [DeepWiki-Open](deepwiki-open/) | Python+TS | 开源代码仓库 Wiki 自动生成平台，Next.js+FastAPI+AdalFlow，9 种 LLM Provider，RAG+Deep Research+Codemap 引用落点 | MIT | [GitHub](https://github.com/AsyncFuncAI/deepwiki-open) |
| [Mem0](mem0/) | Python+TS | AI Agent 智能记忆层：V3 ADD-only 记忆管线 + 多信号检索（语义+BM25+实体），四类 Provider 插件体系，双模式部署 + MCP | Apache-2.0 | [GitHub](https://github.com/mem0ai/mem0) |
| [Memos](memos/) | Go+TS | 自托管笔记应用，微博式时间线，Proto 驱动 API，双重协议（Connect RPC + gRPC-Gateway），三数据库支持，SSE 实时推送，Go+React 工程实践范本 | MIT | [GitHub](https://github.com/usememos/memos) |
| [OpenWork](openwork/) | TS + Electron | 开源 AI 工作流分享桌面应用，Claude Cowork/Codex 开源替代；代理运行时+单一 baseUrl 透明代理，Den 企业控制平面，Drizzle+MySQL 约 80 表，MCP 两工具契约（search/execute），多云 Helm 部署 + 证据卷轴 evals | Fair Source（主体）/ ee 目录 Fair Source | [GitHub](https://github.com/different-ai/openwork) |
| [OpenHands](openhands/) | TypeScript + Electron | 开源开发者控制中心前端（Agent Canvas）：多后端本地/云切换（OpenHands/Claude Code/Codex/ACP），原生 WebSocket 实时会话流，Automations 自动化编排，manifest 安全收纳层，Electron 桌面交付（内置 uv/node 运行时栈），客户端本地存储分层 | MIT | [GitHub](https://github.com/OpenHands/OpenHands) |
| [Tutti](tutti/) | Go + TS (Electron) | 多 AI 智能体实时共享协作空间（"信使"解放）：loopback daemon + SQLite 真值，Tutti Mode Checkpoint 仲裁多 agent，App Center/Factory 应用生态，Connector 市场 + Agent Extension(ACP) 双边接入，signal-and-reconcile 事件流 | Apache-2.0 | [GitHub](https://github.com/anomalyco/tutti) |
| [Goose](goose/) | Rust | Linux Foundation AAIF 通用 AI Agent（桌面/CLI/API）：单主 Agent 循环，统一 Provider trait 接 15+ LLM + 本地推理，自研 MCP client 接 70+ 扩展，SQLite 会话持久化 + auto-compaction，ACP 协议双向打通（服务端被嵌入 + 客户端反接 Claude Code/Codex），Skill/Recipe 体系，工具权限审批 + prompt 注入扫描 | Apache-2.0 | [GitHub](https://github.com/aaif-goose/goose) |
| [AionUi](aionui/) | TypeScript + Rust | 开源多 Agent 桌面 Cowork 平台：app-shell 守护外部 aioncore(Rust) 引擎，ACP 收敛几十种 CLI Agent，Team Mode 协作，Cron 24/7 + WebUI 远程 + IM 渠道，MCP 统一管理（前端壳 + 后端大脑双层架构） | Apache-2.0 | [GitHub](https://github.com/iOfficeAI/AionUi) |
| [AgenticSeek](AgenticSeek/) | Python + TS | 100% 本地 Manus 替代品：双模型投票 Agent 路由（BART+LLM）、LLM 动态改计划的 Planner 编排、Block 代码块工具协议（多语言解释器+safety）、led 摘要模型记忆压缩、stealth 浏览器自主上网/填表、会话持久化 | GPL-3.0 | [GitHub](https://github.com/Fosowl/agenticSeek) |
| [Eigent](eigent/) | Python + TS (Electron) | 开源 Cowork Desktop（Multi-Agent Workforce）：CAMEL-AI 多 Agent 编排（单 agent/Workforce 复杂任务三车道路由）、TaskLock 队列 + SSE 事件流、控制面 server(FastAPI+SQLModel+PostgreSQL) / 行动面 backend 分离、记忆磁盘档案 + 预算投影、MCP(stdio+streamable HTTP) 双写总线、内置浏览器 CDP/终端 PTY、Celery 定时自动化 | Apache-2.0 | [GitHub](https://github.com/eigent-ai/eigent) |
