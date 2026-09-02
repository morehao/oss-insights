# 变更记录

> 本解读对 DeepSeek Harness（`dsh`）的跨版本变更汇总；按版本倒序。项目无官方 changelog，故基于上游 git 代码变动（tag 间 diff / release commit / commit message 聚合）生成。各版本条目的"相比上一版"描述即对应某次增量解读落盘时更新的内容。

## [0.1.2-alpha.4]（2026-09-01 落盘）

> 依据上游 `dsh-v0.1.1-rc.2 → dsh-v0.1.2-alpha.4` git diff（跨 4 个 alpha 版本、含 4 个 release commit）比对生成。

相比 **0.1.1-rc.2** 的主要变更：

**破坏性变更**
- **SQLite 会话持久化后端整体移除**（`refactor(session)!: remove SQLite persistence backend`）：`packages/session/session-persistence-sqlite` 目录删除，事件日志收敛为 JSONL 单一 shipped 后端；Schema 17 打包物理行、`SCHEMA_VERSION` pragma 拒绝随之失效。KV 层 `storage-sqlite` 保留。see [data-model.md](data-model.md)。
- **session 事件 seq 与 log offset 品牌分离**（`refactor(session)!`）：`SessionSeq`（事件身份）/ `SessionLogOffset`（间隙/前缀/读偏移）两个 branded number；`SessionHeader.seedLength` 存储坐标改 `isSeeded: boolean` + body 侧 `inheritedEventCount`；v0 JSONL header 字节兼容（缺席/零/非零 seedLength 解码规则保持）。see [session.md](session.md)。
- **apiproxy RPC 移除**（`refactor(apiproxy)!` ×2）：settings/credentials RPC 与 directory-picker RPC 整体删除（约 −839 行），由 Typert Remote 控制器取代。see [remote-controllers.md](remote-controllers.md)。

**新增能力 / 行为**
- **Typert Remote 控制器体系**：`@Remote`/`@RemoteScope` 声明式暴露 + 构建期 `typert-generator` 生成 Host/Client 双契约；`api-remotes` 挂载、`api-gateway` 经 Connection `/api` 路由分发；`session`/`workspace`/`settings`/`credentials` 四控制器落地；事件经 `API_REMOTE_FORWARDED_EVENTS` 白名单转发。see [remote-controllers.md](remote-controllers.md)。
- **Webhook 子系统**：`ctx.webhookRuntime` 无状态 fire-and-forget 规则运行时 + `webhook-github` 签名 HTTP 适配器；外部投递经规则变成普通 root Session（`source.kind: 'webhook'` 溯源）。see [webhook.md](webhook.md)。
- **Inspector（experimental，private）**：CDP 跨 Host/浏览器双域检视，CDP 状态隔离在 Worker + 收集器快照投影。see [inspector.md](inspector.md)。
- **WebWorker（experimental，private）**：浏览器 Worker 跑整棵插件树（packer 打 VFS 镜像、运行时零编译器、嵌套 Worker 子进程）。see [webworker.md](webworker.md)。
- **storage-json per-record 布局**：`single`（整文件）与 `per-record`（`<root>/<unit>/<table>/<key>.json` + global.json）双布局，legacy 整文件首开自动 bootstrap；投影缓存域随之每会话一文档。
- **projection-cache 冷读播种**：`coldSnapshot` 从身份校验的缓存行 seed fold、尾重放补足、写回刷新（fail-soft）；`checkpointIdentity` 加入 `isSeeded`/`inheritedEventCount` 谱系位，`row.seq` 用 `SessionSeqCursor`。
- **sdk-minimal profile 与 SDK/ACP application bundle**：独立双工具 SDK profile（显式排除 base/Web/settings/凭据/telemetry 等）+ Python/TS SDK 与 ACP 的预置组合分发。
- **web fetch 默认暴露**：base/headless/sdk 三面默认暴露 web fetch，web 形态要求一次性 fetch 审批（one-shot approval）。
- **subagent 模型路由**：`tool-subagent` 新增 model-selection 状态与设置面，可为 Codex/Claude Code provider 配置模型，模型路由可经 DSH SDK 透传。
- **session 持久化模型选择**：`session` 持久化并共享模型选择目录；session-query 新增 shared projected observations。
- **DeepSeek 适配层**：增量 session 日志上传、插件包元数据上传、会话上传评审反馈（session-log-deepseek / deepseek-llm-api-extensions）。

**Web 前端 UX**
- **全会话 turn rail**：session-turn-outline 整日志 turn 大纲投影 + ui-chat 侧栏 load-and-jump 深度分页 + settled-response 预算预览。
- **lexical composer**：ui-conversation 用词法编辑器取代 textarea 栈；连接恢复指示器；usage/time 双 pill；大滚动性能优化（流式几何采样节流、帧节流发布等）。

**接口 / API 变更**
- `api-gateway`（旧 apiproxy）的 `ctx.remote.<namespace>` 调用面取代手工 RPC；`ctx.webhookRuntime` 为新增 capability；`ctx.inspector` 为 experimental 注册面。
- session 构造器对 seed 显式要求（含空 seed + cut 0），`ownEvents()`/`isOwnSeq()` 隐藏继承 cut 比较。

**移除 / 回滚**
- **format migration decoder pipeline 引入后回滚**：`feat(session): add format migration decoder pipeline`（cc9ab200c7）在 alpha 周期内被 `Revert`（211e6939e3），不进入 alpha.4。
- `session-persistence-sqlite`、`tool-subagent-report`、旧 apiproxy 的 credentials/settings/rpc 实现被移除。

## [0.1.1-rc.2]（2026-08-23 落盘）

> 依据上游 `dsh-v0.1.1-rc.1 → HEAD(0.1.1-rc.2)` git diff（含 release commit `aa6c361a97`）比对生成。

相比 **0.1.1-rc.1** 的主要变更：

**新增能力 / 行为**
- **图片/附件管线统一为"确定性归一化 + 对象投影"**：上传图片落库前被归一化为固定 8-bit sRGB、去元数据与动画（仅"干净的源"字节级透传，其余重编码 + 降采样到 `DEFAULT_NORMALIZED_IMAGE_MAX_DIMENSION=2048` 长边）；`AttachmentStore` 新增抽象 `readImageRequest` 投影能力，按模型路由的像素/字节预算派生确定性请求图（`RequestImageAttachment`，本地实现带磁盘 cache 与 variant 身份）。see [attachments.md](attachments.md)。
- **master 与 Files API 请求管线合并（file-id 首选、base64 兜底）**：DeepSeek 适配器请求图片默认走 `file`（Files API 上传复用），超限量化移除；仅当 Files 解析失败或预算超限才回退内联 base64，并对 provider 拒绝/过期 file-id 做失效重试。
- **新增 DeepSeek Files API transport**：完整 `/files` 端点客户端（upload/list/retrieve/delete）、带过期时间与并发去重的上传索引、对 `dsh-` 前缀自有文件按 `reclaimOldestOwned` 回收以应对配额。
- **新增 `prepareCall` 两阶段调用 API**：adapter 从单一 `stream` 演进为 `prepareCall(provider, model)` 产出"冻结的模型信息 + stream 函数"，使同一模型/连接快照绑定、可独立重放；跨 adapter 一致（llm-deepseek / llm-pi-ai 均导出）。see [llm.md](llm.md)。
- **Files 与 stream 超时解耦**：新增独立 `filesApiTimeoutMs`（默认 60s）配置，与 stream idle 超时（300s）分离；Files 解析失败自动回退 base64。
- **`read_image` 工具上报降采样维度与坐标换算提示**：读图结果携带 `originalDimensions`，model-facing envelope 给"multiply coordinates by …"。

**Schema / 数据**
- `SCHEMA_VERSION` **未变化**（仍为 17），session 持久化格式保持稳定。

**接口 / API 变更**
- **权限默认继承回滚**：整体回滚此前合入的 `reuseWorkspaceBlank`（空白会话复用继承默认权限），`session.create` 移除 `reuseWorkspaceBlank`+`sessionId` 组合参数，sessions-port 接口与 UI 权限展示同步收敛。

**移除 / 修正**
- `packages/image` 代码收敛进 `packages/attachment`，attachment-local 新增 `normalization.ts` / `request-image.ts` / `compression-limiter.ts` / `encoding.ts`。
- WebP alpha 兼容修正：sharp 从全不透明 WebP 省略 alpha 平面时不再误判失败。

## [0.1.1-rc.1]（2026-08-23 落盘）

> 依据上游 `dsh-v0.1.0-rc.8 → dsh-v0.1.1-rc.1` git diff（含 release commit `3ec5e8f8c4`）比对生成。

相比 **0.1.0-rc.8** 的主要变更：

**新增能力 / 行为**
- **凭据系统从"环境变量引用"升级为"可交互授权 + 持久化记录"**：新增 `packages/credentials/authorization` 子系统（`AuthorizationPrompt` 的 text/secret/select 三态人机提问、`AuthorizationMethod` 描述 OAuth/API-Key），provider 可注册授权 flow；新增持久化 `CredentialRecord`（`ApiKeyRecord | GrantRecord`，grant payload 对 harness 不透明）、两段式 `CredentialKey = <scope>/<id>` 键空间。see [credentials.md](credentials.md)。
- **凭据文档升级为版本化布局**：`$DSH_HOME/.credentials.yaml` 引入 `DOCUMENT_VERSION=1` 结构化布局（`{version, refs, records}`），boot 时 `migrateFlatDocument()` 一次性迁移旧平面前置布局，写回持有跨进程 writer lock、外部编辑热发布。pi-ai 提供商现在能真正"登录"（OAuth/device-code 桥接进通用授权平面）。
- **Session 投影重写为"状态 + 客户端视图"两段式**：`ProjectionDefinition` 改为声明式纯 fold（`stateSchema` 校验持久化状态、可选 `wire` 提供客户端视图）；新增 host-only 单元（状态不进客户端快照仍照常 checkpoint）；新增统一 `checkpoint()` / `restoreFloor()`，跨 boot 稳定快照。see [session.md](session.md)。
- **DeepSeek 视觉模型进目录**：新增模型 `deepseek-v4-flash-vision-exp`（`inputModalities: ['text','image']`），模型目录新增 `inputModalities` 校验，vision 多模态落地。see [llm.md](llm.md)。

**接口 / API 变更**
- `apiproxy` 的 `session.create` 支持 `reuseWorkspaceBlank` 复用空白会话（配套权限预设方向 `origin` 追踪：`default | selection | inferred`）——**注：该能力在下一个版本 rc.2 被回滚**。
- `web server` 新增结构化 `webserver/index-inject` 事件与 `collectIndexInjections()`，取代裸字符串 `tapIndex` 注入；前端缺失路径返 404、非 GET/HEAD 返 405。

**安全 / 沙箱**
- bwrap profile 新增 `--unshare-pid`，把沙箱进程放进独立 PID 命名空间，关闭 process-root 逃逸。see [sandbox-capability-seams.md](sandbox-capability-seams.md)。

**Web 前端 UX**
- 子代理会话头部升级为"血缘切换器"（`SubagentHeaderLineage`，openChild/refresh/catalog）；同回合重试耗尽渲染"终态 turn 错误"行；宽表按列数自适应 + hover 滚动条；`ask_user_question` 支持多行回答（上限六行）。
