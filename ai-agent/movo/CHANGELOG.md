# 变更记录

> 本解读对 MOVO（Community Edition）的跨版本变更汇总；按版本倒序。上游不维护面向社区版的详细 changelog，故以本地仓库的 tag/提交与代码快照为依据。

## [首次解读] v0.2.0@c6f20f6

依据：v0.2.0@c6f20f6（全新模式首次解读，无上一版本可对比）

本条目建立解读基线。`select-baseline.mjs` 按「稳定 tag > 稳定分支 > 默认分支」的顺序选定 semver 最高的稳定 tag `v0.2.0`（`c6f20f6`），工作树干净、就地检出到 detached HEAD 完成全部解读，收尾已恢复 `main`。本次为**全新解读**，产物为项目 `README.md` + `architecture.md` + 14 篇关注面子文档 + 本文件。

### 覆盖范围（新增能力）

- `README.md`：项目定位与根因、概览与核心能力清单、技术栈、架构总览、子文档索引、核心模块导读、总结与风险。
- `architecture.md`：DSH 与 Python 两个半场的职责切分、内核契约与唯一实现、12 服务拓扑与双网络边界、冷启动链路、一次对话的跨服务时序、事件归一化与双轨投递、Runtime Profile 编译面、五个核心选择及其代价。
- `runtime-host.md`：把 DSH 当精确锁版本的 npm 依赖装载（两级 `createRequire` 解析 `dsh-base` / `dsh-web-app` 的 bundle）、官方 `appBoot.boot` + 空根配置 + 按 entry id 对齐的幂等覆盖层、六个不改内核源码的兼容层插件、REST 与 NDJSON 事件面、模型与工具反向代理、`isolationKey` 多 runtime 隔离、进程内 journal 与游标恢复、DSH 版本准入闸门、Windows 两类锁清理兜底。
- `session-and-turn-lifecycle.md`：`claim_turn` 在 binding 文档上的原子准入、白名单化 `turnContext`、native cursor 与 `stream_seq` 双游标、`DshEventMapper` + `KernelEventProjector` 的两级归一化、`TurnEventChannel` 的 live 直投与 durable 落库、终态不抢跑与 finally 补发、崩溃恢复与三段式取消、会话持久化的 seq 契约、上下文压缩（40 条触发 / 保留 20 条 / `context_summary`）、会话共享的完整保真 ADR 与其六条已知限制。
- `runtime-profile-and-model-gateway.md`：管理配置到不可变 Runtime Profile 的编译与哈希版本、secret-free schema 硬约束、每租户单一 active 版本的发布语义、换密钥与换模型身份的两条生效路径、模型推理反向代理与自签短期 token 的三方 scope、工具 schema 方言转换与可见性 seam、凭据租约的刷新时机与失败语义、桌面端引导复用同一份快照。
- `skill-system.md`：SKILL.md 与 ZIP 两种权威、打包与安装的三道入口与硬限制（5 MB 归档 / 20 MB 展开 / 256 文件 / 压缩比 200:1）、编译期发现与轮内选择、三级按需披露、包内资源读取沙箱、用量从事件流观察、草稿-发布-分发三层生命周期、SkillHub 安装路径、多租户治理口径。
- `tools-mcp-approval.md`：内核侧工具描述符与 `execute` 回调、pre-execute 策略判定、审批请求携带参数、幂等键与收据、工具清单的三条来源与编译期裁剪、不可变快照与签名白名单、服务端重推导授权、MCP Streamable HTTP 一次性会话与 schema 落差处理、两条互不重叠的审批轨道（桌面代理与企业审批状态机）、四本审计账与信任边界。
- `knowledge-and-rag.md`：admin-api 作业回调驱动的 `document-parser`（Docling）解析、两段式分块、向量只写 Weaviate 而出处与授权只留 MongoDB、检索后按文档级策略批量过滤、`document_id` + `chunk_id` 复合引用回传、上下文预算配置存在而消费缺失的现状。
- `content-generation.md`：报告与 PPTX 的蓝图双渲染、DOCX/XLSX/PDF 的最薄库落地、制品签名与投递范围、Word 合同评审这一「规则与模型分工」的垂直 Agent 样本及其未实现边界。
- `browser-agent.md`：服务端控制面加桌面端执行体的分离架构、`browser_task` 工具契约、WebSocket 本地代理桥、`browser_resume` 挂起恢复协议、`suspend` 释放锁与 `resume` 起新轮的语义、企业审批面与仓库内外边界。
- `research-and-evidence.md`：多轮渐进式研究的编排权归属、搜索 provider 的运行时解析与去重、执行期证据包与回合级证据槽、对用户裁剪与对写作模型换载体的双投影、引用坐标保真与覆盖度核算、外网载荷清洗、知识库 RAG 与研究链路的边界。
- `data-model.md`：`main_id` 作为租户键的选择、`chat_sessions` / `chat_messages` / `session_participants` / `agent_kernel_bindings` / `kernel_event_inbox` / `kernel_event_projections` / `runtime_profile_versions` 的字段与索引、唯一索引 `(main_id, session_id, seq)` 的并发语义、两种序号的区别、单轮 turn 的完整写入路径、加法优先的 Schema 演进与 `sequence_repair` 停写式修复、UI 历史的三条来源合并。
- `tenancy-quota-billing.md`：`main_id` 显式传参与查询层收敛、默认租户兜底带来的静默泄漏面、edition 开关关闭计费但保留配额、token 用量按说话者归因、定时任务的执行用户与静默失败边界、`billing.py` 中的遗留死代码。
- `deployment.md`：12 个 Compose 服务拓扑与 `public` / `backend` 双网络边界、bootstrap 一次性密钥与共享卷传参、预构建镜像与版本锁定、`./movo` CLI 的升级/备份/修复路径、健康检查门控与开源发布卫生门禁。
- `frontend-workspace.md`：`user-web` 与 `admin-web` 约 7.5 万行的组织方式、单根组件加模块级运行时 Store（无 vue-router / 无 pinia）的取舍、DSH 事件的 V3 投影与流式渲染、断线重连协议、注册表驱动的富内容交付物、setup wizard 与治理控制台的信息架构。
- `admin-console.md`：admin-api 独立成服务的理由、`ADMIN_BACKEND_SERVICE_TOKEN` 与管理员 JWT 构成的信任矩阵、setup wizard 的 Mongo CAS 原子锁与补偿清理、模型凭据在共享库加共享密钥下的存储与引用式下发、产品扩展点与 edition 判定。

### 关注面判定

- 按 ai-agent 类型规格逐项核对：主 Agent 循环、工具调用、Skill 体系、MCP 接入、会话与任务状态、Runtime、通信机制、数据模型、前端工作区、部署与治理均已成文（落点见 `README.md`「子文档索引」与「核心模块导读」）。
- **主 Agent 循环判定为「由内核承载、不在本仓库」**：规划、上下文装配、工具调用循环均由 DSH 内核实现，MOVO 只做宿主化、边界与治理；因此本解读不设「Agent 循环」子文档，而在 `runtime-host.md` 与 `architecture.md` 中说明这条边界与它带来的代价。
- **记忆机制判定为「分层存在」**：会话级上下文压缩在 `session-and-turn-lifecycle.md`（40 条触发 / 保留 20 条 / `context_summary` 消息），外部知识记忆在 `knowledge-and-rag.md`，跨回合研究结果复用与证据槽在 `research-and-evidence.md`；未发现跨会话的用户偏好画像，故不单列「长期记忆」子文档。
- **ACP 判定为不适用**：MOVO 不对外提供 client→agent 协议，也没有收养外部 CLI Agent；对内只有 chat-api 到 Runtime Host 的私有 HTTP + NDJSON 协议。

### 已知偏差与待修

- Python 侧 `KERNEL_VERSION` 仍为 `0.1.0-rc.6`，而 Node host 声明内核 `0.1.6-alpha.1`；全链路没有运行时版本握手或协议协商，兼容性只靠部署一致性与 CI 准入（见 `runtime-host.md`）。
- 设计哲学中「Kernel 边界化」需打折扣：`AgentKernelContract` 只覆盖 8 个方法，运行时内部仍直接依赖具体类 `DshAgentKernelGateway` 并使用契约之外的 `attach_session` / `events_once`（见 `architecture.md`）。
