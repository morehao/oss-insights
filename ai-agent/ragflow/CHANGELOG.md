# 变更记录

> 本解读对 RAGFlow（infiniflow/ragflow）的跨版本变更汇总；按版本倒序。描述清楚相关变动即可：若上游有官方 changelog，参照它并结合代码变动生成；没有则基于代码变动（git diff / release notes / 代码快照比对）生成。

## [dev-20260902]（2026-09-03 增量）

> 依据：本地源码仓库 `git diff`（`dev-20260826`=`6c2bca8`..HEAD=`4168b3f`，247 commits）。本版为对 dev-20260826 解读的增量更新，同步 Go 侧解析后端的大步演进、连接器扩容与部署面新增开关，并对架构/摄取/部署三篇做深度行号重核。

- **新增能力：Go 侧 in-process native DeepDoc 后端**（`c28edde` feat(deepdoc) #18731，0d1338d 加固）——`-tags cgo` 静态链接 ONNX Runtime（`internal/deepdoc/native/` + `internal/deepdoc/parser/pdf/inference/native_analyzer/`），`cmd/ragflow_server.go` 的 `registerNativeDeepDoc` 在 `api`/`ingestor` mode 启动时按 `NeedsDeepDoc` 强制注册并 fail-fast；`resolveDocAnalyzer` 注释明言原 `DEEPDOC_URL` 选择的外部 Python DeepDoc HTTP 服务已从生产路径与测试套件整体移除（[architecture.md](architecture.md)、[ingestion-deepdoc.md](ingestion-deepdoc.md) 演进说明）。
  - 影响文档：`architecture.md`、`ingestion-deepdoc.md`、`deployment-admin.md`
  - 关键新落点：`internal/deepdoc/native/`、`internal/deepdoc/parser/pdf/inference/native_analyzer/native_analyzer.go`、`cmd/ragflow_server.go:1091` 起、`internal/common/environments.go:214` 起
- **新增能力：同步器连接器净增 9 个**（注册面 29→38）：Box（`f62b10d`）、Asana（`78527d7`）、Airtable（`5217f99`）、OneDrive（`dd8cb0b`）、OCI Storage（`41bc872`）、Zendesk（`90ee505`）、Xquik X search（`0201c1a`）、SeaFile（`8b90c03`）、BigQuery（`c63c237`）；均注册于 `internal/syncer/connector/builtin.go`。
  - 影响文档：`ingestion-deepdoc.md`、`README.md`
- **解析后端**：`paddleocr` 支持自托管 PaddleX 部署（`62028e9`，`PADDLEOCR_BASE_URL`/`PADDLEOCR_ACCESS_TOKEN`）；DeepDoc native 将 OCR `drop_score` 默认统一到 0.5（`DEEPDOC_DROP_SCORE`）。
  - 影响文档：`ingestion-deepdoc.md`
- **Agent 平台**：canvas 的 Iteration 节点暴露 `max_concurrency`（`07660a1`）；Python 侧 `rag/advanced_rag/harness` agentic search 三次重构（`fc62487`/`eb408d8`/`4c1cb64`：删 pipeline.py、新增 tools/compiled_expansion.py 与 structure_qa.py、action_session 大改）——Go `internal/agent/harness/` 图式状态机为另一条独立实现线，本次仅结构稳定小幅改动（+114/-59）。
  - 影响文档：`rag-pipeline.md`（引用行号重核）、`agent-platform.md`（引用行号重核）
- **部署与运维**：默认镜像 `v0.27.0→v0.27.1`；docker-compose 移除独立 `deepdoc` profile 容器；新增 `ADMIN_DEFAULT_PASSWORD`（admin 首启建默认超管，随机密码写 `logs/admin_bootstrap_password.txt`）、`SANDBOX_EXECUTOR_MANAGER_API_TOKEN`、`SANDBOX_CONTAINER_NETWORK`（默认 none）；`sandbox.self_managed` 移除 `pool_size`；ingestor `max_concurrent_workers` 默认 1→2（`de99c75`）。
  - 影响文档：`deployment-admin.md`
- **架构演进注记**：Go 栈由此具备「不依赖 Python 解析服务」的自足能力，但默认交付（`.env` `API_PROXY_SCHEME=python`）仍是 Python 栈。
- **引用行号重核**：architecture/deployment/ingestion 三篇 `verify-references.mjs --diff` 清零 STALE；rag-pipeline 15 处、agent-platform 4 处 STALE 行号同步；agent-platform/data-model 的无指纹历史引用保持原样（未逐条补锚点）。
- 其余大量提交为 fix 级（web/parser/agentic_rag 容错、nats 队列策略 `DiscardNew` 等），不构成用户可感知能力变化，未逐一展开。

## [dev-20260826]（2026-08-26 增量）

> 依据：本地源码仓库 `git diff`（`2db8eb6..HEAD` 区间 + 近期 commit）。本版为对初版解读（基线 `2db8eb6`）的增量更新，重点同步 Agentic RAG 重构与新增数据源连接器，并对全部子文档引用行号做了一次深度重核。

- **Agentic RAG 机制重构**：初版记录的 `internal/agentic_rag/`（ReAct 六工具：think/todo_write/grep_chunks/search_chunks/list_chunks/run_javascript）已被上游 `23a0711`（Revert "port smart-reasoning agentic RAG to eino ADK"）删除；Agentic RAG 改为 `internal/agent/harness/` 下的图式状态机（route → pre_search → planner → orchestrator → formalize_answer，mode 驱动）。
  - 影响文档：`rag-pipeline.md`、`agent-platform.md`
  - 关键新落点：`internal/agent/harness/{agentic_rag,agentic,pipeline,production,types}.go`、`internal/agent/tool/agentic_search.go`
- **新增数据源连接器**：Azure DevOps connector（仓库源码 + 拉取请求，支持 hosted/self-hosted，checkpoint/resume + commit 变更探测）、Azure Blob Storage connector（对象容器，checkpoint 续传）；均注册于 `internal/syncer/connector/builtin.go`。
  - 影响文档：`README.md`、`ingestion-deepdoc.md`
- **引用行号深度重核**：对全部 7 篇 md 重跑 `verify-references.mjs`，覆盖漂移行号（MISMATCH）、修正失效路径；`internal/agentic_rag/*` 五处失效引用改写为 `internal/agent/harness/*` 新路径。
- 其余大量近期提交为 fix 级（PDF 解析、web 样式、agent 容错、memory prompt 对齐等），不构成用户可感知的能力变化，未逐一展开。
