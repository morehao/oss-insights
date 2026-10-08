# 变更记录

> 本解读对 RAGFlow（infiniflow/ragflow）的跨版本变更汇总；按版本倒序。描述清楚相关变动即可：若上游有官方 changelog，参照它并结合代码变动生成；没有则基于代码变动（git diff / release notes / 代码快照比对）生成。

## [v1.0.0-rc1]（预发布 tag）

> 依据：`v1.0.0-rc1`@`14bb02ee`（上一版依据 `v0.27.2`@`a024bea0`，区间 **578 commits、2939 文件、+203029 / −481683 行**，其中 **1225 个文件删除**）。上游有官方 release notes（`docs/release_notes.md`），本版参照它并结合代码变动生成。
>
> **基线性质声明**：`v1.0.0-rc1` 是**用户显式指定的预发布 tag，非稳定基线**。skill 的基线优先级为「稳定 tag > 稳定分支 > main/master」，预发布 tag 不在其列，故本档案所有结论均标注此前提，引用与数值以该 tag 工作树为准。

**交付形态置换：Python 服务栈删除，收敛为 Go 单栈（本版最大变更）**
- 官方 release notes 首条即声明 “comprehensive rewrite of RAGFlow in Go”（`docs/release_notes.md:16-18`），消息队列 Redis→NATS、缓存与 checkpoint 统一落 Kvrocks。
- **删除范围**（1225 文件，按顶层目录）：`test/` 521、`rag/` 192、`api/` 142、`common/` 109、`agent/` 71、`deepdoc/` 65、`internal/` 52、`helm/` 19、`admin/` 19、`memory/` 13、`web/` 7、`tools/` 5、`mcp/` 3、`docker/` 3。**`deepdoc/`、`memory/`、`mcp/` 顶层、`admin/` 顶层、`helm/` 已完全删除**。
- **删除由 9 个提交分阶段完成**：`562e0218`（Remove python source code，23 文件）→ `670e6887`（part2，505 文件，−205285）→ `efbb8007`（−81356）→ `53c4f55f`（part2，−97830）→ `adf2e0c0`（移除 py 测试用例，166 文件）→ `e111e1f9`（移除 Python Dockerfile/compose）→ `c2f17b77`（重写 entrypoint，4 文件）→ `a9c73040`（移除死掉的 deepdoc_oss 服务）→ `d64b84c7`（web 侧移除 python/hybrid 代理配置）。顺序即**源码 → 测试 → 交付脚本 → 前端配置**，每步保持仓库可构建。
- **交付文件收敛**：`docker/entrypoint.sh`、`Dockerfile_go`、`Dockerfile_deepdoc_oss`、`docker/docker-compose-go.yml`、`docker/.env-go` 全部删除；`docker/entrypoint-go.sh` 成唯一 entrypoint，**全文 0 次出现 `API_PROXY_SCHEME`**（`docker/entrypoint-go.sh:137-140` 硬编码 `cp -f ragflow.conf.golang`）；仓库根 `Dockerfile` 即原 `Dockerfile_go`（stage 名 `go-builder`，`:270` `ENTRYPOINT ["./entrypoint-go.sh"]`）；`docker/.env:263-265` 默认值翻转为 `API_PROXY_SCHEME=go`、`python` 被注释。
- **Python 余量分类**（全仓 186 个 `.py`）：资产约 24（`rag/prompts`、`rag/res`、`api/db/init_data/compilation_templates`，由 Go 读取）；**服务 42（`agent/sandbox/`，独立容器 `sandbox-executor-manager`，`profiles: [sandbox]`，绑 `127.0.0.1:9385`，挂 docker.sock）**；工具与客户端约 106（`test/` 77、`tools/` 24、`sdk/` 19、`example/` 5、`scripts/` 2 等）。`sdk/python/` 保留并升到 `1.0.0-rc1`。
- 影响文档：`architecture.md`、`go-rewrite-migration.md`（新建）、`README.md`

**存储层换血：NATS JetStream + Kvrocks**
- 队列唯一后端为 `nats`：`ParseQueueEngineConfig` 只识别 `case "nats"`，其余报错（`internal/server/config/queue_engine_config.go:35-46`）；服务模板 ingestor 的 `mq_type` 写死 `'nats'`（`docker/service_conf.yaml.template:115-116`）；compose 以 `-js -sd /data` 启动（`docker/docker-compose-base.yml:357-366`）。
- cache 后端固定 Kvrocks，理由是不受内存 `maxmemory` 上限约束（`internal/server/config/cache_engine_config.go:27-31`）；配置层把 `"redis"` 与 `"kvrocks"` **映射到同一实现**，`redis` 仅为兼容共享模板残留（`:47-52`）。go.mod 无专用 Kvrocks 客户端，全走 `go-redis/v9`（协议兼容）。
- **checkpoint 三个落点分工**：图执行 → NATS KV（`internal/harness/graph/checkpoint/nats.go`，bucket `checkpoints`，History=3 + `MaxGraphIdle`/`GCInterval` 双层 GC）；canvas → Kvrocks `agent:cp:`（`internal/agent/canvas/checkpoint_store.go:34-36`，`RedisCheckPointStore`→`KvrocksCheckPointStore`）；摄取 per-chunk → Kvrocks（与 RunTracker 同对齐 7 天 TTL）。
- Valkey 服务**就地替换**为 `apache/kvrocks:2.16.0`（`docker/docker-compose-base.yml:298-307`），容器数净减一；但 compose 仍留**孤儿卷** `redis_data`，且 `docker/migration.sh` **逐字节未变**、卷清单缺 `kvrocks_data`/`nats_data`/`clickhouse_data`（`docker/migration.sh:15-16`、`:59-63`）——**用官方脚本备份后再升级存在丢 Kvrocks 数据的风险**。
- MySQL 调优：`innodb_flush_log_at_trx_commit`/`sync_binlog` 放宽、buffer pool 4G→1G（`:246-250`、`:259`），实测摄取吞吐 **23 → ~56 docs/min**（`:232`）。
- 影响文档：`architecture.md`、`go-rewrite-migration.md`、`deployment-admin.md`

**检索管线实质变更**
- **非 Infinity 融合权重改变**：`"0.05,0.95"` → `const esFusionWeights = "0.001,1"`（`internal/service/nlp/retrieval.go:636-639`），注释说明留极小词面权重只为避开"零权重被拒"的引擎限制；首轮召回因此变成**纯向量 pass**，精排统一交下游 `RerankWithKNN`。Infinity 分支保持 `vector_similarity_weight`（默认 0.3）动态权重（`:608-614`）。遗留：`FusionExpr` 注释仍写旧值 `0.05,0.95`（`internal/engine/types/types.go:131-135`）。
- **citation 加厚为三段式**：`internal/service/citation.go` 从约 360 行涨到 **669 行**，新增 `RepairBadCitationFormats`/`RepairSlotCitations`/`ExpandRangeCitations`/`ResolveCitationMarkers` 四个修复器（`ResolveCitationMarkers` 见 `:531-532`）；`CitationPrompt` 新增"禁止把逐个引用合并成 `[ID:i-j]`"的硬约束（`internal/rag/prompts/generator.go:119-120`）。Deep Researcher 有独立 `kbPrompt`（`internal/service/deep_researcher.go:716-717`）与调用点（`internal/service/ask_service.go:277`）。
- **`RERANK_TOKEN_LIMIT_MODE` 简化**：Go 侧不再对 `ErrRerankTokenLimitPolicy` 做 `errors.Is` 特判上抛，改由 `RerankByModel` 原样透传（`internal/service/nlp/reranker.go:168-170`）。三态解析仍在 `internal/entity/models/types.go:388-402`，哨兵 `:15-18`，默认 `defaultMaxRerankTokens = 8196`。
- **`parser_config` 反转**：`graphrag`/`raptor` 由上一版的"被剥离"改为**显式允许**（`internal/service/dataset/helpers.go:206-208`），并新增结构化校验（`:289-290`：`use_graphrag` 须 bool、`entity_types` 须字符串列表、`method ∈ light|general|ner`）；`validIndexTypes = ["graph","raptor","mindmap"]`（`:53-54`）。
- 三条检索 API 全部 Go 路由，前两条共用同一 handler（`internal/router/router.go:334`、`:373-374`）；Python `chunk_api.py` 删除，hybrid nginx 不再回路由 Python（`docker/nginx/ragflow.conf.hybrid:63-64`）。
- **`use_kg` 变为"三通一堵"**：chat（`chat_pipeline.go:986-987`）、Dify（`dify_retrieval_handler.go:293-294`，本版新增）、deep researcher（`deep_researcher.go:394-395`）已打通；REST `/datasets/search` 仍 warn-and-skip（`dataset/search.go:325-327`）；canvas 工具与 NLP adapter 仍拒绝且文案过期（"请使用 Python Canvas"，而 Python Canvas 已不存在）（`agent/tool/retrieval.go:36-38`、`retrieval_nlp.go:204-205`）。
- 影响文档：`rag-pipeline.md`、`data-model.md`

**Agent 平台与能力面**
- **Agentic RAG 换家**：`internal/agent/harness/`（34 文件）删除，迁至**新增的顶层包 `internal/rag/`**（79 个 `.go` / 约 +55,929 行）；研究图 7 节点、会话图 3 节点双层 eino 图，预算集中 `internal/rag/agentic-rag/policy.go:53-55`。
- **顶层 `internal/harness/` 不是本版新增**：v0.27.2 已存在同构的 264 个 `.go`，本区间**仅改 2 个文件**（`README.md` +1/−1、`events/nats.go` +19/−9）；全仓**零生产 import**，属未接线框架。此项纠正了上一版解读把 Agentic RAG 归因于它的错误。
- **记忆重做为持久化状态机 + 租约抢占**：新增 `memory_task` 表（`pending→extracted→stored→completed/failed`，含 `next_retry_at`/`attempt_count`/`lease_owner`/`lease_expires_at`/`last_error`，`internal/entity/memory_task.go:24-28`），`MemoryTaskDAO.Claim` 用 CAS 取代 Redis stream 消费者竞争；NATS `tasks.RAGFLOW` 载荷退化为 `{task_id, task_type}` 唤醒信号（`internal/common/task.go:24-37`）。记忆分层澄清为 **raw/semantic/episodic/procedural 四类位掩码**（非上一版的"语义+程序"两分）。
- **MCP 内→外原地重写**为官方 `github.com/modelcontextprotocol/go-sdk v1.8.0`，声明 4 个协议版本；`types.go`（225 行）删除，新增 `http.go`（138 行，`Options{SSE,StreamableHTTP,JSONResponse}`）与 `tools.json`（schema 外置 `//go:embed`）。仍是 3 工具（retrieval/list_datasets/list_chats），**无 resources/prompts**。外→内仍自研 `internal/utility/mcp_client.go`，仅 `sse`/`streamable-http`，**无 stdio**。新增 Agent 组件级绑定 `buildAgentMCPTools`（`internal/agent/component/agent_mcp.go:21-23`）。
- **Skill 为独立新子系统且与 Agent 零耦合**：`skill_spaces` 表（三态 active/deleted/deleting）、`SKILL.md`(YAML frontmatter)、ES/Infinity 独立索引（`conf/skill_es_mapping.json`，7 档 `q_*_vec` 向量）、`SkillSearchService.hybridSearch` 混合检索、`/v1/skills/*` REST、CLI 从 `local|github|clawhub|skillssh` 安装并安全扫描。**但 `grep -rni "skill" --include=*.go internal/agent/` 命中 0**——Skill 不是 Agent 组件/工具，无系统提示注入；唯一"Skill→系统上下文"实现位于未接线的 `harness/core/middlewares/skill/`。另存在同名的第二概念（知识编译产出的 skill 页面，`/datasets/:id/skills`，底层 `compile_kwd = skill/skill_all`）。
- **两处功能回退**：①渠道**不能再绑定 Agent**——Go `ChatChannel` 无 `agent_id` 列（`internal/entity/chat_channel.go:20-29`），`HandleIncomingMessage` 只认 `chat_id`（`internal/service/chat_channel.go:296-298`），而 v0.27.2 的 Python `chat_channel` 表有该列并有 `validate_agent_target`；②**无 ACP / Agent Client Protocol / agent server**（全仓 grep 0 命中），`internal/cli/` 只是 HTTP 管理客户端。
- 沙箱：Go 侧仍是客户端抽象，7 provider 顺序含 `local`（`internal/agent/sandbox/catalog.go:16-18`），新增 **Tenki 云沙箱**（`go.mod:14`，`github.com/LuxorLabs/tenki-sdk-go/sandbox v0.7.0`）、`catalog.go`(397 行)/`probe.go`(107 行)/`testdata/admin_contract.json`(653 行) 三份新文件；执行体仍在 Python `agent/sandbox/`（Go 从未重写沙箱运行体）。
- `internal/agent/canvas/stream.go`+`stream_test.go`（253 行）删除，SSE 帧格式化下沉到 `internal/service/bot_completion.go:201`。
- 影响文档：`agent-platform.md`、`rag-pipeline.md`

**Schema 与数据演进**
- 实体 **49 → 54 表：仅新增 5 张，零删除**：`conversation_message`、`conversation_reference`、`api_4_conversation_message`、`api_4_conversation_reference`、`memory_task`。
- 会话历史拆子表：`ChatSession` 通过 `gorm:"-"` 排除 `message`/`reference`（`internal/entity/chat.go:66-74`）；`conversation_message` 用 `(conversation_id, position)` 复合主键 + CASCADE（`internal/entity/conversation_history.go:34-39`）。
- **迁移顺序反转**：`RunMigrations` 先于 `AutoMigrate`（`internal/dao/database.go:249-254`），保护 `tenant_model.model_type` 不被强转 text→int。
- **无 `migration_version` 表**：版本存 `system_settings` 的 `mysql_migration.database.version`（`internal/dao/migration_version.go:36-39`）。
- 新增 3 个迁移文件：`parser_config_migration.go`（TokenChunker:SixApplesFall → GeneralChunker:SixApplesFall，仅 `parser_id ∈ {general,naive}` 且 `pipeline_id` 为空，批次 256）、`conversation_history_migration.go`（目标 `v1.0.0-rc1`，批次 100，可断点续跑）、`model_migration.go`（1556 行，移植 `tools/scripts/mysql_migration.py`，把模型 id 改写为 `<model>@default@<provider>`）。
- `document.run` 列已从 Go 实体移除但仍以 `HasColumn` 兜底引用（幽灵列）；`internal/dao/database.go:405-416` 在非 migrate 模式自动建 **7** 张 Go 专属运行时表。
- 索引映射：8 个文件在 `conf/`，其中 `os_mapping.json` 与 `message_infinity_mapping.json` 是**孤儿**（零引用）；引擎派发仅 elasticsearch/infinity/oceanbase+seekdb/serenedb，**无 opensearch**（`internal/engine/global.go:53-65`）。`*_with_weight` 模板语义改变：`text + index:"false"` → `keyword + index:true + ignore_above:10000 + similarity:boolean`，模式加宽为 `^.*_(with_weight|list)$`（`conf/mapping.json:132-140`）。
- 影响文档：`data-model.md`、`go-rewrite-migration.md`、`deployment-admin.md`

**摄取与解析**
- **断点续传语义重写**：eino 图级 checkpoint **整体移除**，`Pipeline.Run` 声明 "always runs the full graph from the entry node"、"Resume-from-checkpoint is intentionally not supported"（`internal/ingestion/pipeline/pipeline.go:182-185`）；续传等价物改为 per-chunk 缓存（`internal/ingestion/pipeline/cleanup.go:11-13` 明说 checkpoint store 已移除，重跑从冷 chunk cache 开始）。`defaultCheckpointTTL` → `defaultTrackerTTL`(7d)，仅用于 RunTracker 状态 hash（`:180`）。
- **run identity 成为一等概念**：`ensureRunIdentity` 是唯一创建路径，发布消息**之前**由 API 侧调用，document 行锁串行分配 `MAX(run_count)+1` 并绑定新建 `pipeline_operation_log` 行（`internal/service/ingestion_task_service.go:760-762`）——取代已删除的 `IncrementRunCount`/`ClearComponentProgress`。
- `processMessage` 删除，状态路由收敛进 `handleAndExecute`（`internal/ingestion/service/ingestion_service.go:447-449`）；`supersedeAndRetryTask`/`forceTerminalTask`/`ownsTaskRun` 全部消失，新增启动恢复 `ScheduleCreatedTasks`（`internal/service/ingestion_task_service.go:690-692`）。
- **`general` 模板切分组件从 `TokenChunker` 换成新增的 `GeneralChunker`**（`chunker/general.go`，1203 行，v0.27.2 无此文件），chunker 变体 4 → **10**（`wire.go` 注释仍写 4，属漂移）。
- 连接器 **40 处/39 类 → 42 处/41 类**（新增 `feishu_wiki`、`zotero`；`google-drive`/`google_drive` 为同一实现别名），`RegisterBuiltIns` 见 `internal/syncer/connector/builtin.go:25-32`。
- 新增推理并发预算 `RAGFLOW_DEEPDOC_INFERENCE_CONCURRENCY`（CLI > env > config > 默认 4，`internal/common/environments.go:259-263`），新文件 `internal/deepdoc/native/inference_limit.go`。权重仍是 FlatBuffer `.ort` 五件（`internal/common/environments.go:282-288`），构建期静态链接 ORT（`build.sh:40-47`）。
- `internal/ingestion/compilation/extractor/`（Python `rag/graphrag/ner` 的 Go 移植）**当前是死代码**：全仓 0 处 import，且仅 `//go:build cgo_thincner` 下编译。
- 影响文档：`ingestion-deepdoc.md`、`rag-pipeline.md`、`go-rewrite-migration.md`

**部署与 Admin**
- 主 compose **只剩 `ragflow-cpu`**（58 行，`ragflow-gpu` 删除），端口仅 80/443/9380/9381；MCP 改为 9380 进程内（`docker/docker-compose.yml:21`）；`command: --enable-adminserver --init-superuser`，`--init-model-provider-tables` 被注释（`:24-27`）。
- **`kvrocks` 就地替换 `redis`/valkey**，profile `ragflow-go`，镜像 `apache/kvrocks:2.16.0`，自渲染 entrypoint `docker/kvrocks-entrypoint.sh`（新增）；`COMPOSE_PROFILES` 追加硬编码 `ragflow-go,clickhouse`（`docker/.env:52`）。
- 端口收敛：nginx→admin **9381**（原 9383）、nginx→API **9380**（原 9384）；`GO_HTTP_PORT`/`GO_ADMIN_PORT` 从 `.env` 删除。
- **Admin 远程 shutdown 在 OSS 未实现**：`ShutdownService`/`StartService`/`RestartService` 返回 "shutdown service not implemented"（`internal/admin/service.go:1152-1159`），`ShutdownIngestor` 调用被注释（`internal/admin/handler.go:1148-1155`），`case <-ShutdownCh` 分支不可达（`internal/ingestion/service/ingestion_service.go:1336-1339`、`internal/syncer/syncer.go:146-153`）；`CleanupStaleStatuses` 是死代码（`internal/admin/state.go:103-113`）；`state.go:43-64` 只认三种 `ServerType`，故 `deepdoc` 无法注册。
- `/healthz` 聚合 5 项，JSON 键仍字面 `"redis"`（`internal/service/system.go:72-80`），ClickHouse 未纳入。心跳间隔为配置 `general.heartbeat_interval` 默认 **3s**，存活窗口 **45s**（`internal/admin/service.go:927`），但 `:926` 注释仍写 "less than 5 seconds"——**不一致持续存在**。
- `runDeepDoc` 是 OSS no-op（`cmd/deepdoc_server_ee.go:24-27`）；`server.StartServer/ShutdownServer` 是 EE stub（`internal/server/server_ee.go:40-53`）。
- `DEEPDOC_URL` 整体移除，in-process native 是唯一生产后端（`docker/entrypoint-go.sh` 不再起 deepdoc 容器）。
- 影响文档：`deployment-admin.md`、`go-rewrite-migration.md`

**API 与 CLI**
- **兼容层整段删除**：`api/apps/backward_compat.py` 随 `api/apps/` 整体删除，Go 路由**无 remap/redirect 中间件**；19 个已移除别名在 `docs/references/http_api_reference.md:68-86` 列表化。唯一双栈可观测手段是响应头 `X-API-Source: go`（`internal/router/router.go:143-147`）。
- 一处文档/代码边界模糊：alias 表称 `/v1/file/*` 不由 Go 后端注册，但 `/v1/file/root_folder|parent_folder|all_parent_folder` 仍注册在非 `/api` group（`internal/router/router.go:445-452`），同族能力同时存在 `/api/v1/files/*`（`:430-433`）两套路径；`internal/handler/file.go:137` 的 swagger 注释仍写已废弃路径。
- **CLI 100% 重写**：`admin/client/ragflow_cli.py` 删除，Go 实现 `internal/cli/`（41 文件，含 `filesystem/` 虚拟文件系统与 `skill_hub/`）+ 入口 `cmd/ragflow-cli.go`（带 `//go:build ignore`，由 `build.sh` 显式构建）。默认 API base `http://<host>:<port>/api/<version>`（`internal/cli/http_client.go:65`）。命令面比 Python 版更宽。
- 影响文档：`go-rewrite-migration.md`、`agent-platform.md`

**已知问题（官方声明 vs 代码核实）**
- 官方三条 known issues（`docs/release_notes.md:33-36`）经逐条核实：**DeepDoc 仅 CPU 成立**（`build.sh:284-286`）；**本地 sandbox 仅文档口径成立**（`internal/agent/sandbox/local.go` 是完整实现、`catalog.go:16-18` 以 `"local"` 打头，实际变化是 compose `profiles: [sandbox]` 默认不启动）；**Team/Me 在代码层面不成立**（`internal/entity/dataset.go:37-41`、`internal/service/team_permission.go:9-12`、`internal/service/dataset/update.go:97-102`、`internal/service/agent.go:1096-1100`、`internal/router/router.go:275-281` 五端点 1:1 对应 Python）。
- 上游自身留有多处未同步文本：`AGENTS.md:14`、`pyproject.toml` 的 `[tool.setuptools] packages`、`docker/README.md:3`、`docker/launch_backend_service.sh` 的 Python 分支（调用已删除的 `tools/scripts/run_migrations.sh`）、compose 里指向 `entrypoint-go.sh` 的死挂载、`ADMIN_DEFAULT_PASSWORD`（无 Go 代码读取，Go 读 `DEFAULT_SUPERUSER_PASSWORD` 而 `InitDefaultAdmin` 直接硬编码 `admin`，`internal/admin/service.go:1475-1479`）、`admin_bootstrap_password.txt`（仅存在于注释）、`serverArgs.adminHost/adminPort`（解析后无读取点，`cmd/ragflow_server.go:85-86`）。
- `internal/rag/agentic-rag/harness/` **目录不存在**，但 `internal/agent/chat/message_fit.go:32` 仍有指向它的注释——失效注释。

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
