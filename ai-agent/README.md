# AI 产品（AI Product）

本目录收录 AI 产品领域的开源项目解读——凡是**完整产品形态、用户可直接使用**的 AI 应用均归此类（不限于桌面端，含 Web、CLI、协作平台、Agent 本体等）。

| 项目 | 语言 | 说明 | 协议 | 仓库 |
|------|------|------|------|------|
| [OpenHuman](openhuman/) | Rust + TS | 本地优先 AI 超级助手桌面应用：进程内 Rust 核心（22 域族/85 模块），分脑编排（triage 四分类 + checkpointed tinyagents graph），记忆树+Obsidian/TinyCortex 记忆引擎，per-call 成本与模型路由，approval-gated 工作流，多链钱包，MCP 协议 | AGPL-3.0 | [GitHub](https://github.com/tinyhumansai/openhuman) |
| [OpenWorker](openworker/) | Python+TS+Rust | 开源 AI 同事桌面平台，本地 sidecar 架构，审批门控 Agent 循环，5+ LLM Provider，26+ 连接器，MCP 协议，技能系统，上下文自动压缩，SSRF 防护 | MIT | [GitHub](https://github.com/andrewyng/openworker) |
| [Lework](Lework/) | Go+TS+Electron | 企业级 AI 工作协作平台（AI 劳动力管理），双进程事件驱动，多 Agent 运行时，Task+Asset 模型，Skill/Tool + MCP/插件，企业级权限/审计 | Apache-2.0 | [GitHub](https://github.com/insmtx/Lework) |
| [Multica](multica/) | Go+TS | 托管式 AI Agent 协作平台，Agent 作为第一公民的任务管理系统 | Apache-2.0 | [GitHub](https://github.com/multica-ai/multica) |
| [OpenBee](openbee/) | Go+TS | 自托管 24/7 数字员工管理平台，对话驱动，多 IM 平台集成，动态引擎路由，多用户 RBAC，49 条 SQLite 迁移 | MIT | [GitHub](https://github.com/theopenbee/openbee) |
| [OpenVort](openvort/) | Python+TS | 开源 AI 员工平台/研发工作流引擎，手写 Agentic Loop，插件化架构 + Slot Protocol，四级 Skill 体系，MCP Server，深度绑定 IM + 项目管理/代码/CI/CD 全流程 | AGPL-3.0 | [GitHub](https://github.com/openvort/openvort) |
| [OpenWork](openwork/) | TS + Electron | 开源 AI 工作流分享桌面应用，Claude Cowork/Codex 开源替代；代理运行时 + 单一 baseUrl 透明代理，Den 企业控制平面，Drizzle+MySQL 约 80 表，MCP 两工具契约，多云 Helm 部署 + 证据卷轴 evals | Fair Source | [GitHub](https://github.com/different-ai/openwork) |
| [OpenCode](opencode/) | TypeScript | 开源 AI 编码 Agent，服务化架构（CLI/TUI/Web/Desktop），四轴可组合 LLM 路由，事件溯源 Session V2，Effect-TS 函数式编排 | MIT | [GitHub](https://github.com/anomalyco/opencode) |
| [OpenHands](openhands/) | TypeScript + Electron | 开源开发者控制中心前端（Agent Canvas）：多后端本地/云切换，原生 WebSocket 实时会话流，Automations 自动化编排，manifest 安全收纳层，Electron 桌面交付，客户端本地存储分层 | MIT | [GitHub](https://github.com/OpenHands/OpenHands) |
| [Tutti](tutti/) | Go + TS (Electron) | 多 AI 智能体实时共享协作空间：loopback daemon + SQLite 真值，Tutti Mode Checkpoint 仲裁多 agent，App Center/Factory 应用生态，Connector 市场 + Agent Extension(ACP) 双边接入，signal-and-reconcile 事件流 | Apache-2.0 | [GitHub](https://github.com/anomalyco/tutti) |
| [Goose](goose/) | Rust | Linux Foundation AAIF 通用 AI Agent：单主 Agent 循环，统一 Provider trait 接 15+ LLM，自研 MCP client 接 70+ 扩展，SQLite 会话持久化，ACP 协议双向打通，Skill/Recipe 体系，工具权限审批 + prompt 注入扫描 | Apache-2.0 | [GitHub](https://github.com/aaif-goose/goose) |
| [AionUi](aionui/) | TypeScript + Rust | 开源多 Agent 桌面 Cowork 平台：app-shell 守护外部 aioncore(Rust) 引擎，ACP 收敛几十种 CLI Agent，Team Mode 协作，Cron 24/7 + WebUI 远程 + IM 渠道，MCP 统一管理 | Apache-2.0 | [GitHub](https://github.com/iOfficeAI/AionUi) |
| [AgenticSeek](AgenticSeek/) | Python + TS | 100% 本地 Manus 替代品：双模型投票 Agent 路由、Planner 编排、Block 代码块工具协议、记忆压缩、stealth 浏览器自主上网/填表、会话持久化 | GPL-3.0 | [GitHub](https://github.com/Fosowl/agenticSeek) |
| [Eigent](eigent/) | Python + TS (Electron) | 开源 Cowork Desktop（Multi-Agent Workforce）：CAMEL-AI 多 Agent 编排（单 agent/Workforce 三车道路由 + TaskLock 队列/SSE），控制面/行动面分离，记忆磁盘档案，MCP 双写总线，浏览器/终端工具，Celery 定时自动化 | Apache-2.0 | [GitHub](https://github.com/eigent-ai/eigent) |
| [WeKnora](weknora/) | Go | 企业级 RAG 与知识推理平台：文档导入→语义索引→Agent/反应式 Agent 推理→自动生成 Wiki+知识图谱，多源连接器（飞书/Notion/语雀/RSS），26 LLM + 7 向量库 + 10 IM 集成，多租户 RBAC，网页挂件/桌面端部署 | MIT | [GitHub](https://github.com/Tencent/WeKnora) |
| [PicoClaw](picoclaw/) | Go | 极致轻量 Go 原生 AI Agent 网关，<10MB 内存跑在 $10 硬件，20+ 通道 30+ LLM，MCP 集成 | MIT | [GitHub](https://github.com/sipeed/picoclaw) |
