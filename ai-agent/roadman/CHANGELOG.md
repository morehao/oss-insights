# 变更记录

> 本解读对 RoadMan 的跨版本变更汇总；按版本倒序。上游未维护官方 changelog，故以本地仓库的 git 提交与代码快照比对为依据。

## [首次解读] main@4196f22

依据：main@4196f22（全新模式首次解读，无上一版本可对比）

本条目建立解读基线。上游仓库无 tag、无官方 changelog，`select-baseline.mjs` 按「稳定 tag > 稳定分支 > 默认分支」矩阵回退到默认分支 `main`，工作树干净、无需检出。本次为**全新解读**，产物为项目 `README.md` 与 11 篇关注面子文档 + 本文件。

### 覆盖范围（新增能力）

- `README.md`：项目定位、根因、双层架构与状态所有权、22 节点工作流总览、一次完整请求链路、模块职责映射、核心模块导读、设计哲学四条主线、同类定位对比（与 kb 内 TripStar 的逐维对比）、不足与风险、可借鉴设计。
- `planning-workflow.md`：LangGraph 装配与节点职责分组、两处条件边（validate/verify）的语义、RoadManState 状态所有权、复核-修复循环与 3 轮上限、best_effort 交付取舍、completed/awaiting_clarification/failed 三态。
- `semantic-agent-layer.md`：LLM 提供商适配层（`LLM_API_STYLE` 双协议）、结构化输出容错与再修复、Prompt 分工与事实约束表达、模型调用审计与隐私边界、无关键词兜底的取舍。
- `tourism-research.md`：多源研究到候选规范化的流水线、同景区变体合并与同名异地身份校验、评分与选取、逐日编排约束（当日路线实际经过、晚间回酒店基点）、适配复核与未知态处理、特殊事件与季节性。
- `routing-and-transport.md`：`preferred_mode` / `allowed_fallback_modes` 语义、`ROUTE_UNAVAILABLE` 传播链、跨城班次多源去重择优、返程闭环时间窗阈值、真实 geometry 与前端示意线的分工。
- `ev-energy-model.md`：SOC 六字段连续账本、补能量三级策略与 `calculation_basis`、每日驾驶上限拆分与跨天衔接、风险提醒与复核接口、续航误差评测与 `--require-real` 语义。
- `skill-registry.md`：SkillAdapter 契约与 Registry 调用路径、缓存 TTL 与审计字段取舍、三类降级的差异化处理、MCP 客户端（Streamable-HTTP / SSE 双解码 + 短生命周期会话）、车型目录多源字段级合并。
- `trip-editing.md`：PlanPatch 数据模型与累积语义、`interpret → preview → apply/reject → rollback` 全链路、影响预览与重算门禁、版本备份与回滚粒度、编辑来源的统一管道。
- `data-model.md`：实体关系（trips / trip_versions / planning_state / jobs / vehicles / files / skill_calls / users / auth_sessions）、存储选型与 JSON 文档权衡、各核心表字段表、Alembic 迁移链与 Schema 生成、读写路径与多用户隔离/级联硬删除。
- `async-runtime.md`：ARQ 任务生命周期与 `planning/start` → worker → SSE 全链路、状态与投影的真相归属、SSE 事件契约与安全边界、队列与并发语义、ops 指标与健康检查。
- `deployment.md`：Compose 五服务拓扑与健康检查门控、前后端镜像多阶段构建与构建期密钥注入、Nginx 反代 SSE 要点、配置分组与凭据治理、备份恢复与排障入口。
- `verification-and-evaluation.md`：测试分层（单元/集成/端到端验收/E2E）、三类评测（需求理解、异常降级、续航误差）的断言与报告形态、模拟数据标注与 `--require-real` 拒绝路径、离线可持续性。

### 关注面判定

- 按 ai-agent 类型规格逐项核对：主 Agent 循环/编排、工具调用、Skill 体系、MCP 接入、会话与任务状态、Runtime、通信机制、数据模型、可部署服务均已成文（落点见 `README.md`「关注面覆盖说明」）。
- **记忆机制判定为不适用**：无跨行程用户记忆/偏好画像（`users` 表无偏好列，偏好是每次请求的输入字段）；唯一名为 memory 的 `MemorySkillCache` 是外部调用结果缓存。
- **宿主接入协议（ACP）判定为不适用**：项目是自带 Web 工作台的最终产品，不对外提供 client→agent 协议，对外只有 HTTP 与 SSE。

### 解读中确认的现状与缺口

以下为**首次解读的现状描述**（非版本变更），记录以便后续增量核对：

- **单文件承载过多**：`backend/app/planning/graph.py` 7700 余行、`planning/tourism.py` 3100 余行、`skills/carinfo.py` 2600 余行，节点实现与启发式逻辑、外部适配混排，可测试性与后续拆分成本偏高。
- **水平扩展受限**：进度与 SSE 依赖单实例可见性，worker 无并发/优先级控制与任务取消入口。
- **启发式权重缺少外部基准**：POI 选取与补能估算的权重、阈值多来自工程经验；评测覆盖续航误差，但对"选点合理性"缺少可比基准。
- **续航评测默认模拟数据**：`evaluation/range_observations.simulated.json` 为回放样本；真实道路数据需通过 `--input <real.json> --require-real` 提供。
- **多用户能力为 Demo 级**：具备账户、Cookie 会话与数据隔离，但缺少隐私告知、授权撤回与数据主体请求流程（项目文档亦如此自述）。
- **文档与实现同步靠人工纪律**：仓库自带近二十份设计/运维文档，未见自动校验文档与代码一致性的机制。
