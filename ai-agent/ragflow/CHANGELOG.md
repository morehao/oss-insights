# 变更记录

> 本解读对 RAGFlow（infiniflow/ragflow）的跨版本变更汇总；按版本倒序。描述清楚相关变动即可：若上游有官方 changelog，参照它并结合代码变动生成；没有则基于代码变动（git diff / release notes / 代码快照比对）生成。

## [v0.27.2]（2026-09-17 增量）

> 依据：`v0.27.2`@`a024bea0`（上一版依据 `dev-20260902`=`4168b3f`，区间 197 commits）。上游有官方 release notes（`docs/release_notes.md` 的 v0.27.2 段），本版参照它并结合代码变动生成；同时把「解读基线」从 dev 快照切到最近稳定 tag。

**Agentic RAG 重构（上游 v0.27.2 头条）**
- 官方 release notes 把 Agentic RAG 检索框架重构列为 v0.27.2 头条（#19046 / #19112 / #19172 / #19209 / #19424 / #19459）。本区间内的落点全在 **Python `rag/advanced_rag/`**：`rag/prompts/action_run.md` 新增 **TOOL PLAYBOOK**（工具组合链、收敛规则、tool-status→next-action 表），`harness/action_session.py` 把 7 个工具 spec 重写为 WHEN TO CALL / DO NOT CALL / ARGUMENTS / OUTPUT / IF IT FAILS 契约并把共享证据池封顶 60（与 SCA view cap 对齐），`agentic_rag_graph.py` 在证据池达上限时短路 SCA 路由直接 finalize（`2bf80b92`）。
- 研究循环健壮性：prose 回复不再被切成子问题（形状守卫拒绝答案/引用样式行）、SCA（自检 agent）超时改为「本轮标记不足并继续」而非接受未验证草稿、slot-table 分解重试预算放宽（`4e3cd423`）；移除 `graph.ainvoke()` 的整图超时，使 final answer 流式阶段不被研究预算打断，研究阶段各节点超时与 180s deadline 保留（`d0e22f4d`）。
- navigation 工具调整（`a11c27e0`）：`harness/tools/navigation.py`、`compiled_expansion.py` 与 `api/apps/services/dataset_api_service.py` 的导航 API 同步扩容。
- **Go 侧 `internal/agent/harness/` 本区间零改动**：Go 图式状态机与 Python `rag/advanced_rag/` 是两条并行实现线，本版性能收益落在 Python 线。
- 影响文档：`rag-pipeline.md`（Agentic RAG 节演进说明）、`README.md`

**新增能力**
- **Sitemap 数据源连接器**（`371cfabd`，`internal/syncer/connector/builtin.go`）：按 sitemap.xml 抓网页入库，带 PDF-pass checkpoint（`14fa5597`）；内置连接器注册项 39→40（去重 `google-drive`/`google_drive` 别名后为 38→39 类）。官方 release notes 同版还列出 WebDAV connector 自定义 CA 证书。
- **MonkeyOCRv2 解析后端**（`fd9f9e7f`）：新增 provider `MonkeyOCRv2-Parsing`（`internal/entity/models/monkeyocrv2.go`、`conf/models/monkeyocrv2.json`），走原生同步 ZIP 协议；MinerU 后端名对齐现行 API（`792a17cf`）、MinerU chart block 不再被静默丢弃（`0c28d59e`）。
- **EPUB 文档预览**（`14306f42`，含编码归一化）。
- **Sofya 联网搜索工具**（`c44e171d`，Python `agent/tools/sofya.py`）：返回整页正文而非摘要，depth（basic/snippets）、top N 与 per-call 参数可配；DuckDuckGo 工具在 news 渠道下改走新闻检索（`293de5aa`）。
- **新模型 provider**：Hubris（`4731ee33`）、llmman（`c051f405`）。
- 交互：知识图谱展示总/当前节点数、图谱搜索回车按实体名高亮节点、检索测试与搜索命中高亮改由前端正则实现（`a1261c81`）、相似度阈值按百分比显示、Team 设置表格命中高亮。
- 影响文档：`README.md`、`ingestion-deepdoc.md`、`agent-platform.md`

**接口与 API 变更**
- **三个检索 API 合并为一个实现**（`b7e577ce` #19303）：Python 侧 `/datasets/search`、`/datasets/<id>/search`、`/retrieval` 收敛到 `api/apps/restful_apis/chunk_api.py` 的同一 handler（路径不变）；Go 侧新增 `POST /api/v1/retrieval`（`internal/router/router.go`），旧 `/datasets/search`、`/datasets/:dataset_id/search` 保留并存；hybrid nginx 把 `/datasets/search` 交回 Python。
- Go 检索请求契约统一（`internal/service/dataset_types.go`）：新增 `page_size`、`document_ids`、`metadata_condition`，`doc_ids`/`use_kg` 保留为兼容别名；`search_id` 增加租户归属校验，chunk 字段做 `chunk_id→id`/`content_with_weight→content`/`kb_id→dataset_id` 重映射。
- 新增 **`RERANK_TOKEN_LIMIT_MODE`**（`a1359309` + `2284c8e1`）：`truncate`（默认，按 `maxTokens - queryTokens` 逐文档截断）/ `passthrough` / `raise_error`；Go 侧解析在 `internal/entity/models/types.go`，越界策略错误 `ErrRerankTokenLimitPolicy` 会向上传播。
- Agent 检索工具接受 `user_id` 节点参数（`a41d0d3a`），用于 memory 检索的 user 过滤，语义对齐 Python `Retrieval._retrieve_memory`。
- 安全收紧：团队共享 chat/agent 会话改为**只读**（`9538c282`）；数据集/工作区 commit 路由补鉴权（`fb2f8818`）；connector rebuild 校验「KB 可访问 + connector 已绑定该 KB」（`d29a4fc2`）；SearchBot 检索前逐 KB 鉴权（`f8fc1301`）；KB 权限判定改 **fail-closed**（`cbaf0669`，`permission != team` 一律拒）；用户置为 inactive 时作废其 AccessToken（`a997a1bd`）；用户/文件 id 改用随机 UUIDv4（`330b988b`）。
- 影响文档：`rag-pipeline.md`、`data-model.md`、`deployment-admin.md`

**Schema 与数据演进**
- `chat_channel` 新增 `agent_id` 列（`24c115bf` #19307）：渠道可直连某个 Agent 画布，配套 `chat_channel_service`/`api_service.get_latest_agent_channel_session`。
- 摄取「排队中」状态透出（`471dd91e` / `255b000f`）：`ingestion_task` 新增复合索引 `idx_ingestion_task_dataset_status(dataset_id, status)`，`document` 列表新增 `ingestion_status` 投影列（join 最新一条 ingestion_task），新增 `CountActiveByDatasetID` 与 `GetByDocumentID` 的确定性排序。
- `InitDB` 在不带 `--migrate` 启动时新增 `autoMigrateRuntimeModels` 兜底建 `ingestion_task`/`ingestion_task_log`（失败仅 Warn）。
- `internal/entity/models`：`RerankModel.MaxTokens`、模型响应体读取限额（正常 16MiB / 错误体 1MiB）、`tool_choice` 支持对象形式。
- **`internal/dao/migration.go` 本区间零变更**：以上结构变更全部由 GORM `AutoMigrate` 生成，无新增手工迁移。
- 影响文档：`data-model.md`

**构建与部署**
- 默认镜像 `v0.27.1` → `v0.27.2`。
- **知识编译运行时设置**（`2476e2fb` #19254，配套 `447f87d7` #19156 限流）：wiki 映射/精炼池、页结构编译批大小、知识图谱批 token 上下限、共享 LLM 池限流重试等 20+ 个旋钮集中进 `docker/.env`。
- Python `task_executor` 支持 `MAX_TASKS_PER_WORKER` / `RECYCLE_SHUTDOWN_TIMEOUT` 定期回收 worker 进程，释放长跑累积的 ONNX Runtime / CUDA arena 碎片（`eb244256`，默认关闭，由 entrypoint supervisor 重启）。
- 修正容器内 ClickHouse 原生 TCP 端口 `9900 → 9000`（`0cee507c`；宿主机映射口仍是 9900）。
- Go 镜像把建表责任收回：`entrypoint-go.sh` 不再执行 Python 迁移脚本、`syncer/admin/api` 一律显式带 `--migrate`；`entrypoint.sh` 的 `ensure_db_init` 从 webserver 分支前移为全局执行（`c5d9295c`）。
- `Dockerfile_go` stage 重排（web/go 构建器统一基础镜像、删独立 cpp-builder、Go module 缓存前移）并预置 ORT 静态库与 `.ort` 权重；`build.sh` 的 ORT 链接从 `--whole-archive` 改为 `--undefined=OrtGetApiBase` + `--dynamic-list`，恢复算子裁剪（`4c9768be` / `d5a89c32` / `aaed9fd4`）；Go DeepDoc 本地推理权重从 `.onnx` 切到 FlatBuffer `.ort`（`6de46a23`）。
- `docker/launch_backend_service.sh` 支持 `--debug`（`4ab91c70`）。
- 影响文档：`deployment-admin.md`、`ingestion-deepdoc.md`

**移除与废弃**
- **任务创建不再产生 GraphRAG / RAPTOR 分支**（`51da9778` #19194）：取任务时从 `parser_config`/`kb_parser_config` 剥离 `graphrag`/`raptor` 键；REST 层默认解析模板不再产出这两段，并新增 `strip_graphrag_raptor_config`（`6ec9c6a7` #19161）。
- 但**代码与检索路径并未删除**：`rag/graphrag/` 本区间零 diff 仍在树中，Python `use_kg` 检索仍可达（`chunk_api.py` → `settings.kg_retriever`）；Go 侧 `use_kg` 是「有字段、无实现」的兼容占位（`internal/service/dataset/search.go` 打 Warn 后跳过），Agent 工具直接返回 `ErrGraphRAGNotSupported`。RAPTOR 实现也仍散落在 `rag/utils/raptor_utils.py`、`rag/svr/task_executor_refactor/raptor_service.py`、`internal/ingestion/component/knowledge_compiler/tree/raptor.go` 等处，被移除的只是 `parser_config.raptor` 这条自动触发通路。
- 知识编译不再把全量实体/关系写成 graph chunk（`3dbca958`）。
- 影响文档：`rag-pipeline.md`、`ingestion-deepdoc.md`、`data-model.md`

**引用与口径重核**
- 按新基线 `v0.27.2` 全量重跑 `verify-references.mjs`：6 篇有引用的文档共 **314 条 `路径:行号` 引用，全部 OK**（WEAK/MISMATCH/UNVERIFIED 均为 0）；16 个 mermaid 块全部通过 `check-mermaid.mjs`。
- 机制性修补：**取消「引用行内联 anchor」写法**（`…（[path:1-2]()<!-- anchor: … -->）…`）。内联 anchor 会先于「引用行之后」的 anchor 被配对器消费，导致同一行多个引用的指纹整体错位级联——这是本档案多篇文档此前出现 MISMATCH/UNVERIFIED 的根因。现统一改为「引用行 + 紧随其后的独立 anchor 行」，并把单行引用数压到 **≤2**（超出则拆段，因为配对器只向前看 2 行）。
- 锚点覆盖：`architecture` 5→11、`data-model` 8→38、`ingestion-deepdoc` 45→86、`rag-pipeline` 73→97、`agent-platform` 25→70、`deployment-admin` 26→64；并对「首/末行是孤立 `}` 或纯标点」的区间按可引用行回缩，使每个区间两端都是可定位的真实代码行。
- 修正若干历史口径错误：①Go 内置摄取模板实为 **12 个**（`internal/ingestion/pipeline/template/ingestion_pipeline_*.json`），此前记为 13 个（把 `knowledge_compiler` 组件误计入模板）；②PDF `parse_method` 白名单实为 **9 项**且含 `monkeyocrv2`，其中带空格的是 `tcadp parser`（此前写 `tcadp`）；③同步器连接器注册面为 **40 处调用 / 39 类源**（此前单说 38）；④`agent-platform.md` 曾描述「`api`/`ingestor` 按 `servermode.NeedsDeepDoc` 注册 DeepDoc 后端」——该包在 v0.27.2 已被删除，注册内联进 mode switch，已改正。
- 区间边界约定：区间两端必须是「可定位的真实代码行」。当某构造的首/末行是孤立的 `}`、`)` 或空行时，区间**向内回缩到最近的实质代码行**（因此部分区间不再包含尾括号），以保证 anchor 任意时刻都能被程序定位；这是本档案此后新增引用时应遵循的同一约定。
- 补齐全部 7 篇解读文档的 YAML frontmatter（`type`/`title`/`description`/`tags`），并在 [README.md](README.md) 概览处补上 `解读基线：\`v0.27.2\`（\`a024bea0\`）`。

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
