---
type: 开源项目次要支撑模块解读
title: Multica 次要支撑层解读——认证授权、计量计费、遥测、issue 支撑与横切基础设施
description: Multica 认证授权、CLI/Public API、计量计费（entitlement/seatcapacity/quota）、analytics 成本管道、agentconfig、issue 支撑、attribution 审计、中间件/可观测/特征开关、scheduler 基础设施、工具库共 11 组次要模块的设计要点与核心链路关系。
tags: [multica, secondary-modules, auth, billing, analytics, scheduler, attribution]
---

# Multica 次要支撑层批量解读

> 本解读只覆盖"次要支撑模块"，深度适中：每组提炼定位 + 关键设计点 + 与核心链路的关系，不展开全流程。引用均来自实际 read，格式为 `仓库相对路径:行号`。

## 读取范围与方法

- 只读研究，未修改任何源码。
- 迁移版本号以 `server/migrations/` 下文件名为准（约 474 组 up/down，编号至 445+）。
- 相关既有模块可能已被其他批次数签出（attribution 部分与 data-model 存在交集，本次仍作轻量覆盖并标注）。

## 组一：auth —— 认证授权（邮件验证码 / JWT / PAT / task token / 分享链接）

**一句话定位**：以「邮件验证码 + 用户名即密码」为主的 auth 层，产出三类长期凭证（JWT、`mul_` PAT、`mdt_` daemon token）与两类运行时凭证（`mat_` task token、`mcn_` cloud PAT），并把「谁是 actor」以权威头注入下游。

**关键设计点 1：验证码登录是自实现的全链路，含频率限制与白名单门控。** 服务端生成 6 位随机码，写入 `verification_code` 表并带 `attempts` 计数（迁移 010 追加），校验用常量时间比较防时序侧信道；发送前按「每邮箱 60 秒 1 条」限频；注册开关 `AllowSignup` / 邮箱、域名白名单三层优先级在 `checkSignupAllowed` 收敛，已有用户永远放行登录。

```go
func (h *Handler) checkSignupAllowed(email string, isNewUser bool) error {
	if !isNewUser {
		return nil // existing users always allowed to log in
	}
```
[server/internal/handler/auth.go:235-267]()
<!-- anchor: func (h *Handler) checkSignupAllowed(email string, isNewUser bool) error { … 	return nil -->

```go
	// Rate limit: max 1 code per 60 seconds per email
	latest, err := h.Queries.GetLatestCodeByEmail(r.Context(), email)
```
[server/internal/handler/auth.go:333-334]()
<!-- anchor: 	// Rate limit: max 1 code per 60 seconds per email … 	latest, err := h.Queries.GetLatestCodeByEmail(r.Context(), email) -->

**关键设计点 2：凭证种类靠 token 前缀分流，身份以「服务端权威头」注入，防伪造。** `middleware.Auth` 按 `mat_`/`mcn_`/`mul_`/JWT 四分支裁决：`mat_` task token 由服务端在任务认领时铸造、daemon 注入 agent 进程，中间件据此写死 `X-Actor-Source: task_token` 并覆盖客户端伪造的 `X-Agent-ID/X-Task-ID`；`mcn_` 云节点 PAT 反向代理到 Multica Cloud Fleet 校验。`X-Actor-Source` 是服务端独占信号，任何客户端值在分支前被丢弃——这是"agent 不得伪装成人"的权威机制。

```go
		if strings.HasPrefix(tokenString, "mat_") {
			if queries == nil {
				http.Error(w, `{"error":"invalid token"}`, http.StatusUnauthorized)
```
[server/internal/middleware/auth.go:87-115]()
<!-- anchor: 		if strings.HasPrefix(tokenString, "mat_") { … 			next.ServeHTTP(w, r) -->

**关键设计点 3：Cloud PAT 的「TTL 即撤销延迟」+ 失败语义细分。** `mcn_` 校验结果在 Redis 缓存 60 秒（不缓存负结果），因为上游不可作废缓存键，TTL 本身就是撤销延迟上界；错误分为 `ErrCloudPATInvalid`(401) / `ErrCloudPATUnavailable`(503) / `ErrCloudPATNotConfigured`(401 fail-closed)，云不可达返回 503 让 CLI/daemon 重试而不是扔掉有效 token。

```go
const cloudPATCacheTTL = 60 * time.Second
```
[server/internal/auth/cloud_pat.go:45-45]()
<!-- anchor: const cloudPATCacheTTL = 60 * time.Second -->

**关键设计点 4：普通主键用 UUIDv7 而秘密/租约键不用。** `dbid.NewV7` 铸造 UUIDv7（48-bit 毫秒时间戳 + 随机），使连续插入在 B-tree 上聚簇；但显式声明：**不为 lease/idempotency key/秘密生成 v7**——那些要不可预测性与无内嵌时间戳，留在 `gen_random_uuid()`/v4，DB 侧保留 `DEFAULT gen_random_uuid()` + `COALESCE` 兜底，使库中长期混排 v4/v7。

```go
func NewV7() pgtype.UUID {
	id, err := uuid.NewV7()
	if err != nil {
		slog.Error("dbid: uuidv7 generation failed, falling back to the database default", "error", err)
```
[server/pkg/dbid/dbid.go:51-58]()
<!-- anchor: func NewV7() pgtype.UUID { … 	return pgtype.UUID{Bytes: [16]byte(id), Valid: true} -->

**关键设计点 5：task token 替代「把 daemon 主人的 PAT 注入 agent」的历史错误。** 迁移 108 注释点明旧实践让 agent 获得主人级权限（MUL-2600）；现在 daemon 认领时铸造短生命周期 `mat_` token，绑定 `(task_id, agent_id, workspace_id, user_id)` 四元组，owner-only 端点据此 403 agent 流量。

```sql
CREATE TABLE task_token (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
```
[server/migrations/108_task_token.up.sql:14-27]()
<!-- anchor: CREATE TABLE task_token ( …     created_at TIMESTAMPTZ NOT NULL DEFAULT now() -->

**与核心链路的关系**：auth 是任务编排的准入闸门——`mat_` token 让 agent 在运行时以受限 actor 身份执行；`X-Actor-Source` 是 attribution（组七）与 human-only 权限判定的前提。

## 组二：CLI / Public API —— 机器访问面与对外契约

**一句话定位**：CLI（`multica` 二进制）通过 REST client 访问服务端，`publicapi/v1` 是对外稳定的能力契约（现主要由 Plugin API 消费），两者共享同一套 X-Client-* 身份/能力头。

**关键设计点 1：CLI 是「命令式 REST client + 统一错误分类」，关注 agent 语义。** `APIClient` 携带 `AgentID/TaskID` 注入 `X-Agent-ID/X-Task-ID`（agent 执行上下文）；所有 ≥400 响应收敛为 `*HTTPError`，并靠 `requestUsedTaskToken`（读**实际发出**的请求头，而非 client 字段）区分"401 该重新登录"与"401 是 task token 过期，直接停手"。

```go
func NewAPIClient(baseURL, workspaceID, token string) *APIClient {
	return &APIClient{
		BaseURL:     strings.TrimRight(baseURL, "/"),
```
[server/internal/cli/client.go:187-194]()
<!-- anchor: func NewAPIClient(baseURL, workspaceID, token string) *APIClient { … 		HTTPClient:  &http.Client{Timeout: httpTimeout()}, -->

**关键设计点 2：能力协商用 `X-Client-Capabilities`，属于协议细节而非用户可选项。** CLI 始终声明 `stable_attachment_urls`：要求列表返回稳定下载路径而非 ~800 字符、每次重签的 CloudFront 签名——签名字节会击穿 agent prompt 缓存（MUL-5372），这是把「性能事故」转化为「契约头」的典型。

```go
const clientCapabilities = "stable_attachment_urls"
```
[server/internal/cli/client.go:209-209]()
<!-- anchor: const clientCapabilities = "stable_attachment_urls" -->

**关键设计点 3：Public API v1 用「能力台账」绑定路由、作用域、限流与文档，OpenAPI 测试钉死防漂移。** `Operations` 数组逐条声明 method/path/contract/policy（含 credential kind、scope、risk、audit 状态、限流 profile）；`OperationPolicy` 把授权要求与可观测性要求记在契约旁，由 surface 中间件执行 credential，资源层执行 workspace/scope——「信任边界」被显式建模为 `CredentialKind`。

```go
var Operations = []Operation{
	{Method: http.MethodGet, Path: PathContext, Contract: ContractPluginExtension, Policy: OperationPolicy{Credentials: pluginCredentials, Risk: RiskRead, Audit: AuditNotRequired, RateLimits: pluginRateLimits}},
```
[server/pkg/publicapi/v1/routes.go:57-67]()
<!-- anchor: var Operations = []Operation{ … 	{Method: http.MethodDelete, Path: PathStorageValue, Contract: ContractPluginExt -->

**关键设计点 4：错误面统一为 RFC 9457 problem+json，带稳定 code 与 legacy `error` 别名。** `WriteProblem` 被 auth middleware 与 handler 共享，路由前失败与资源内失败同一契约；`code` 由 status 推导（如 402→`payment_required`），并回写 `Request-ID`。

```go
func WriteProblem(w http.ResponseWriter, r *http.Request, status int, code, detail string) {
	if code == "" {
		code = CodeForStatus(status)
	}
```
[server/pkg/publicapi/v1/problem.go:80-103]()
<!-- anchor: func WriteProblem(w http.ResponseWriter, r *http.Request, status int, code, detail string) { … 	_ = json.NewEncoder(w).Encode(payload) -->

**关键设计点 5：数据迁移用「回填前向修复」，部署即疗愈历史坏账。** 迁移 079 把历史失败任务的 `failure_reason` 重新分类为 `api_invalid_request`，否则 GetLastTaskSession 的 resume 查询会回落到旧 `agent_error` 行、重毒化每次重试（MUL-1921/1918）——迁移不只是改 schema，还承担「部署后修复存量数据」的职责。

```sql
UPDATE agent_task_queue
SET failure_reason = 'api_invalid_request'
WHERE status = 'failed'
```
[server/migrations/079_backfill_api_invalid_request.up.sql:13-18]()
<!-- anchor: UPDATE agent_task_queue …   AND error ILIKE '%invalid_request_error%'; -->

**与核心链路的关系**：CLI 是本地 daemon 与远程服务之间的执行通道（task 认领、agent 上下文注入、结果回传都走 `APIClient`）；Public API 是 Agent/Plugin 生态对外暴露受控能力的最小切片。

## 组三：entitlement / seatcapacity / quota —— 计量计费与额度闸门

**一句话定位**：entitlement 是「Cloud 下发策略、本地缓存执行」的通用闸门；seatcapacity 是「Cloud 预购座位协议」的事务化 outbox；autopilot quota 是它的第一个持久化消费实例。

**关键设计点 1：Provider 接口「无错误返回」——每个失败都归约为一个 fail-open 的 Decision。** 决策枚举 `off/observe/enforce`；缓存命中新鲜→直接 enforce，过期但未出宽容窗口→`observe`（降级观察、不再拒绝新工作），彻底不可用→`off`。接口层注释直言：消费者不需要看错误，只需要一个可审计的决策与原因。

```go
type Provider interface {
	Gate(ctx context.Context, workspaceID uuid.UUID, name GateName) Decision
}
```
[server/internal/entitlement/types.go:75-77]()
<!-- anchor: type Provider interface { … 	Gate(ctx context.Context, workspaceID uuid.UUID, name GateName) Decision -->

```go
	if cachedOK && now.Before(cached.retryAfter) {
		c.recordCache("retry_suppressed")
		if cached.hasPolicy && now.Before(cached.staleUntil) {
			return c.recordDecision(name, decisionFromEntry(cached, name, ReasonStale, true))
```
[server/internal/entitlement/client.go:114-117]()
<!-- anchor: 	if cachedOK && now.Before(cached.retryAfter) { … 			return c.recordDecision(name, decisionFromEntry(cached, name, ReasonStale, true)) -->

**关键设计点 2：stale 的 enforce 策略会被「降级为 observe」再放行——过期数据可以服务但不可拦截。** `decisionFromEntry` 中 `stale && gate.Action == ActionEnforce` 会把 action 改写为 `ActionObserve`；配合 `singleflight` 每工作区一次在飞刷新、以及「策略版本回退拒绝写入缓存」(`version_regression`)，保证永远不用旧版本策略去硬拒新请求。

```go
func decisionFromEntry(entry cacheEntry, name GateName, reason Reason, stale bool) Decision {
	gate := cloneGate(entry.policy.gates[name])
	if stale && gate.Action == ActionEnforce {
		gate.Action = ActionObserve
```
[server/internal/entitlement/client.go:337-341]()
<!-- anchor: func decisionFromEntry(entry cacheEntry, name GateName, reason Reason, stale bool) Decision { … 		gate.Action = ActionObserve -->

**关键设计点 3：seat 协议走「事务化 outbox + 租约化 worker」，多副本不放大同一 backlog。** `seat_capacity_outbox` 只记录跨服务写意图（无商业限额），每行有 `lease_token`/`attempt_count`/`next_attempt_at`/`dead_lettered_at`；worker 每 30s 扫描到期行、以 lease 原子认领后再打 Cloud，死信前最多 10 次、指数退避到 5 分钟。行级 CHECK 保证各 action 必备字段完整。

```sql
CREATE TABLE seat_capacity_outbox (
    workspace_id UUID NOT NULL,
    operation_token UUID NOT NULL,
```
[server/migrations/415_seat_capacity_outbox.up.sql:4-34]()
<!-- anchor: CREATE TABLE seat_capacity_outbox ( …     CHECK ((action <> 'release_member') OR member_id IS NOT NULL) -->

```go
	for i := int32(0); i < w.batchSize; i++ {
		intent, err := w.queries.ClaimNextDueSeatCapacityIntent(ctx, pgtype.Timestamptz{
			Time: w.now().Add(defaultClaimLease), Valid: true,
```
[server/internal/seatcapacity/worker.go:115-159]()
<!-- anchor: func (w *Worker) ReconcileOnce(ctx context.Context) error { … 	return nil -->

**关键设计点 4：配额实现是「策略无关的持久化台账 + CAS 状态机」。** `autopilot_quota_period/reservation` 表不含任何商业默认值（限额/周期由 Cloud 运行时下发）；reservation 以 `idempotency_key` 唯一，状态 `reserved→consumed/released` 走 CAS 转换；达到限额且 enforce 时记录 `blocked_counts` 并拒绝，observe 时只记指标。策略关闭时**完全绕过配额表**走 legacy INSERT，保证自托管零配额依赖。

```go
	wouldBlock := period.UsedCount+period.ReservedCount >= policy.limit
	if wouldBlock && policy.action == entitlement.ActionEnforce {
		if _, err := qtx.IncrementAutopilotQuotaBlocked(ctx, db.IncrementAutopilotQuotaBlockedParams{
```
[server/internal/service/autopilot_quota.go:171-173]()
<!-- anchor: 	wouldBlock := period.UsedCount+period.ReservedCount >= policy.limit … 		if _, err := qtx.IncrementAutopilotQuotaBlocked(ctx, db.IncrementAutopilotQuo -->

**与核心链路的关系**：entitlement 是「人在环上 + 商业边界」的执行点——issue 创建闸门（`GateIssueCount`）与 autopilot 执行闸门（`GateAutopilotRuns`）分别拦在 issue 写入与任务入队之前，是计费与产品可用性的交界面。

## 组四：analytics / task_usage —— 遥测与成本计量管道

**一句话定位**：analytics 是非阻塞的产品遥测（PostHog，现已降级为 metrics-only）；task_usage 是「原始 token 记录 → 小时级 rollup → 权威成本」的成本计量管道，DB 内完成、幂等可重放。

**关键设计点 1：analytics 的「绝不拖慢主链路」纪律，以及 PostHog 服务端事件整体退役（MUL-4127）。** Capture 进入有界队列、后台 worker 批量 flush，队列满即丢并计数；无 API key 时 no-op。演化上更狠的一步：`metricsOnlyEvents` 覆盖了**全部**服务端事件——PostHog 不再接收服务端产品事件，漏斗直接从业务库与 Grafana 计数器读取，PostHog 只剩前端崩溃/错误遥测。

```go
type Client interface {
	Capture(e Event)
	// Close drains pending events. Call once during graceful shutdown.
	Close()
}
```
[server/internal/analytics/client.go:60-61]()
<!-- anchor: type Client interface { … 	Capture(e Event) -->

```go
var metricsOnlyEvents = map[string]struct{}{
	// Product-behaviour events — DB + Grafana are the source of truth
	// (MUL-4127); the PostHog copy was redundant.
	EventSignup:                        {},
```
[server/internal/analytics/events.go:54-83]()
<!-- anchor: var metricsOnlyEvents = map[string]struct{}{ … 	EventAutopilotRunFailed:    {}, -->

**关键设计点 2：task_usage rollup 是「DB 内自愈」的管道，幂等=「全量重算而非增量」。** 101 建 UTC 小时桶 + 单行 watermark 表 + dirty 队列（覆盖 watermark 看不到的 DELETE/cascade/项目改挂）；102 用 4 个 BEFORE 触发器入队 + 一个窗口函数按 dirty key **从原始表 SUM 重建**该桶（非 delta），重跑重叠窗口结果不变，使「cron + 离线 backfill」可安全并发；每个 pipeline 有独立 advisory lock id。

```sql
CREATE TABLE task_usage_hourly (
    bucket_hour         TIMESTAMPTZ NOT NULL,   -- UTC, truncated to hour boundary
```
[server/migrations/101_task_usage_hourly_schema.up.sql:37-55]()
<!-- anchor: CREATE TABLE task_usage_hourly ( …         (bucket_hour, workspace_id, runtime_id, agent_id, project_id, provider, model) -->

```sql
CREATE OR REPLACE FUNCTION task_usage_hour_bucket(ts TIMESTAMPTZ)
RETURNS TIMESTAMPTZ
```
[server/migrations/102_task_usage_hourly_pipeline.up.sql:31-37]()
<!-- anchor: CREATE OR REPLACE FUNCTION task_usage_hour_bucket(ts TIMESTAMPTZ) …     SELECT (date_trunc('hour', ts AT TIME ZONE 'UTC')) AT TIME ZONE 'UTC'; -->

**关键设计点 3：权威成本与估算成本双轨并存，NULL 语义即「未重算」。** 213 加 `cost_usd_ticks`（xAI 以 1e-10 美元 tick 上报，整数端到端精确），rollup 桶同时携带权威成本和 `uncosted_*` token（无权威成本那部分）；`uncosted_*` 列**可空无默认**——NULL 表示「该桶还没按新 schema 重算过」，读者 COALESCE 到整桶 token 总额即等于旧行为，避免历史桶成本突然归零。

```sql
ALTER TABLE task_usage
    ADD COLUMN IF NOT EXISTS cost_usd_ticks BIGINT;
```
[server/migrations/213_task_usage_authoritative_cost.up.sql:21-66]()
<!-- anchor: ALTER TABLE task_usage …     ADD COLUMN IF NOT EXISTS uncosted_cache_write_tokens BIGINT; -->

**关键设计点 4：session 级 advisory lock 泄漏事故的教科书级修复。** 272 把 rollup 的 lock 4246 从 session 级换成 transaction 级（`pg_try_advisory_xact_lock`）：旧版在 `EXCEPTION WHEN OTHERS` 里手动 unlock，但 `OTHERS` 不匹配 `query_canceled`（超时/取消直接跳过 handler），session 锁在随后的回滚中存活，pgxpool 把仍持锁的连接还池 → 后续 tick 全返回 0、usage 聚合静默停止，连 DeleteWorkspace 都永久阻塞（MUL-5983）。tx 级锁随事务结束（含回滚）自动释放。

```sql
CREATE OR REPLACE FUNCTION rollup_task_usage_hourly()
RETURNS BIGINT
```
[server/migrations/272_rollup_task_usage_hourly_xact_lock.up.sql:38-94]()
<!-- anchor: CREATE OR REPLACE FUNCTION rollup_task_usage_hourly() … 	PERFORM prune_task_usage_hourly_dirty(); -->

**关键设计点 5：调度器只做壳，核心函数在 SQL 里自持锁。** `TaskUsageHourlyJob` 的 handler 只读 watermark、调 `rollup_task_usage_hourly()`、再读 watermark 写审计行；函数内部已持 advisory lock，与遗留 pg_cron tick 天然串行——DB-backed 调度（组九）与 DB 内管道锁的整合样板。

```go
func TaskUsageHourlyJob(pool *pgxpool.Pool) JobSpec {
	return JobSpec{
		Name:              JobNameRollupTaskUsageHourly,
		Cadence:           5 * time.Minute,
```
[server/internal/scheduler/jobs_task_usage.go:36-56]()
<!-- anchor: func TaskUsageHourlyJob(pool *pgxpool.Pool) JobSpec { … 		Handler: makeTaskUsageHourlyHandler(pool), -->

**与核心链路的关系**：task_usage 每行由一次任务完成写入（task 链路尾部），rollup 支撑 dashboard/账单；analytics 的 signup/issue_executed 事件是「获取→激活→扩张」漏斗在核心执行路径上的埋点。

## 组五：agentconfig —— Agent 配置面（并发、env、模型、thinking、服务等级）

**一句话定位**：agent 的「运行参数配置面」由逐次演进叠加的列构成——并发上限、自定义环境变量、自定义 CLI 参数、模型、thinking 等级、服务等级，均为「运行时透传给 CLI、服务端不做跨运行时抽象」的存储。

**关键设计点 1：并发上限是服务端强校验的数值域。** `agentconfig` 包定义 `1..50` 合法区间（默认 6），创建/更新 agent 时校验，防止配置漂移到无界并发。

```go
const (
	DefaultMaxConcurrentTasks int32 = 6
	MinMaxConcurrentTasks     int32 = 1
	MaxMaxConcurrentTasks     int32 = 50
)
```
[server/internal/agentconfig/concurrency.go:5-9]()
<!-- anchor: const ( … 	MaxMaxConcurrentTasks     int32 = 50 -->

**关键设计点 2：自定义 env/args 是「透传不解析」的 JSONB，运行时拼进子进程命令行。** 迁移 040/041 分别加 `custom_env`/`custom_args` JSONB 列（支持 router/proxy、Bedrock、Vertex 等模式）；服务端不跨运行时抽象化这些值——`thinking_level` 甚至直接存运行时原生字符串（Claude 的 `low|medium|high`、Codex 的 `none|minimal|...`），因为「用户看到的和 CLI UI 宣传的一致」。

```sql
ALTER TABLE agent ADD COLUMN custom_env JSONB NOT NULL DEFAULT '{}';
```
[server/migrations/040_agent_custom_env.up.sql:5-5]()
<!-- anchor: ALTER TABLE agent ADD COLUMN custom_env JSONB NOT NULL DEFAULT '{}'; -->

**关键设计点 3：env 是秘密，读面做最小暴露。** agent 资源响应故意不序列化 custom_env 明文，只暴露 `has_custom_env`/`custom_env_key_count`；明文只在 owner-only 的 `/api/agents/{id}/env` 端点与 CLI env 管理命令可见（与 `mat_` task token 的 owner-only 403 相呼应）。

```go
	// custom_env is intentionally NOT serialized on agent resources. The
	// list response and the agent resource
	// only expose coarse metadata (has_custom_env, custom_env_key_count) so
```
[server/internal/handler/agent.go:84-86]()
<!-- anchor: 	// custom_env is intentionally NOT serialized on agent resources. The … 	// only expose coarse metadata (has_custom_env, custom_env_key_count) so -->

**与核心链路的关系**：这些列在 daemon 启动 agent 子进程时被读入并注入（env/args/thinking/service_tier → CLI 标志），是「配置面」到「执行面」的传递层。

## 组六：issue 支撑 —— status 目录、防重复、排序、properties、views、编号

**一句话定位**：一组把 issue 从「裸行」扩展成可编排产品的支撑：自定义状态目录（行为等价类）、并发安全的防重复闸门、float 排序、JSONB 属性包、保存视图、人类可读编号。

**关键设计点 1：自定义状态用「行为等价类 = category = 内置键」模型，避免「只继承一半行为」的怪胎状态。** `Effective` 是核心转换函数：内置键直接返回自身（零查询、行为字节不变），自定义键经目录解析回 category；解析失败时**返回原键不改动**——fail-safe 方向是「别猜，保持现状」，防止被 sweeper/auto-trigger 误判。7 个 category 是行为等价类（in_progress/in_review/blocked 在 autopilot 终结、delegated 通知等行为上各不相同），因此不做 Linear 式「started」压缩。

```go
func Effective(ctx context.Context, q Querier, workspaceID pgtype.UUID, status string) string {
	if IsBuiltIn(status) {
		return status
	}
```
[server/internal/issuestatus/issuestatus.go:283-298]()
<!-- anchor: func Effective(ctx context.Context, q Querier, workspaceID pgtype.UUID, status string) string { … 	return entry.Category -->

**关键设计点 2：目录派生键是「AI/CLI 友好的 ASCII handle」，非拉丁名称回退 category+序号。** `DeriveKey` 把显示名 slugify 成 `^[a-z0-9][a-z0-9_]{0,31}$` 的键（`issue.status` 存的就是它，CLI 参数也是它）；名字不含拉丁字母时回退 `<category>_2`，让 agent 至少能从键猜出行为邻居；archived 状态也占键位（部分索引之外的普通唯一索引），防重名冲突。

```go
	if slug := slugify(name); slug != "" {
		if IsBuiltIn(slug) {
			return "", fmt.Errorf("%q is a built-in status key and cannot be reused; rename the status or pass an explicit key", slug)
```
[server/internal/issuestatus/issuestatus.go:199-213]()
<!-- anchor: func DeriveKey(name, category string, taken map[string]bool) (string, error) { … 	return firstFreeKey(category, taken) -->

**关键设计点 3：防重复是「先拿 DB 行锁、再查、再插入」的并发安全闸门。** `LockAndFindActiveDuplicate` 先 `LockIssueDuplicateKey`（按 workspace/project/parent/normalized-title 组成的锁键），未勾 `allow_duplicate` 时把 Done/Cancelled 之外的重复拦截，报错文案直接教用户 `allow_duplicate=true`；autopilot 版本额外加了时间窗口。

```go
	normalizedTitle := NormalizeTitle(title)
	if normalizedTitle == "" {
		return db.Issue{}, false, nil
	}
	if err := q.LockIssueDuplicateKey(ctx, lockKey(workspaceID, projectID, parentIssueID, normalizedTitle)); err != nil {
```
[server/internal/issueguard/duplicate.go:45-86]()
<!-- anchor: func LockAndFindActiveDuplicate( … 	return duplicate, true, nil -->

**关键设计点 4：排序用 float 的「顶部插入 = MIN(position)−1」，无浮点序号耗尽问题。** 手排列新项落在列顶（position ASC 语义），与 UpdateIssue 内联策略同源，改一处必须同步另一处。

```go
	).Scan(&minPos); err != nil {
		return 0, fmt.Errorf("query min issue position: %w", err)
	}
	return minPos - 1, nil
```
[server/internal/issueposition/position.go:21-30]()
<!-- anchor: func NextTopPosition(ctx context.Context, q queryRower, workspaceID pgtype.UUID, status string) (float64, error) { … 	return minPos - 1, nil -->

**关键设计点 5：properties 是「定义目录 + JSONB 值包」，重命名零迁移。** `issue_property` 定义目录（archived 不删），issue 上 `properties` JSONB 以**定义 UUID 为键**，因此定义改名不触发值迁移；GIN 与容量 CHECK（≤16KB）分属后续单语句迁移（CREATE INDEX CONCURRENTLY 纪律）。

```sql
CREATE TABLE issue_property (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL,
```
[server/migrations/191_issue_properties.up.sql:15-40]()
<!-- anchor: CREATE TABLE issue_property ( … ALTER TABLE issue VALIDATE CONSTRAINT issue_properties_size_limit; -->

**关键设计点 6：issue 编号是「前缀 + 计数器 + 唯一约束」的人类可读 ID。** 迁移 020 回填 `issue_prefix`（工作区名前 3 大写字母）与按 created_at 的序号，加 `UNIQUE(workspace_id, number)`——`{prefix}-{number}` 成为 CLI 与分享引用里的稳定标识。

[server/migrations/020_issue_number.up.sql:1-36]()
<!-- anchor-base64:LS0gQWRkIGlzc3VlX3ByZWZpeCBhbmQgaXNzdWVfY291bnRlciB0byB3b3Jrc3BhY2UgZm9yIGh1bWFuLXJlYWRhYmxlIGlzc3VlIElEcy4g4oCmIENSRUFURSBJTkRFWCBpZHhfaXNzdWVfd29ya3NwYWNlX251bWJlciBPTiBpc3N1ZSh3b3Jrc3BhY2VfaWQsIG51bWJlcik7 -->

**与核心链路的关系**：issuestatus 是任务编排中「状态语义」的判官（autopilot 终结、sweeper、inbox 通知都问 `Effective`）；issueguard 防重复位于 issue 创建/agent 自生成入口；position/properties/views 全是 UI 编排能力。

## 组七：attribution / 审计 —— 人类归因

**一句话定位**：让每次 agent run 都能追溯到一个「可解释的负责人类」，且归因只做可见性/审计/成本，**永不做权限判定**（MUL-4302）。

**关键设计点 1：归因是「词汇表 + 纯分类规则」分离架构。** 包内定义 Source/EvidenceKind/TriggerKind 三套词汇；分类函数 `ClassifyComment/ClassifyDirect/DirectHumanRun/RuleOwner/TriggerOwner` 只接收调用方已取好的事实（side-effect free、可单测），DB 读取留在 service 层。`Source` 瀑布包括 direct_human → delegation（复制不链式，防循环）→ comment_source → trigger_owner（优于 rule_owner，谁武装了触发谁负责）→ owner_fallback（降级）。

```go
type Source string

const (
	// SourceDirectHuman — a member's own action enqueued the run (comment,
```
[server/internal/attribution/attribution.go:29-69]()
<!-- anchor: type Source string … 	SourceUnattributed Source = "unattributed" -->

**关键设计点 2：单向不变量「originator 存在 ⟹ accountable = originator」在应用层收敛 + DB 层兜底。** `finalizeAttribution` 是唯一收缩点：UserID(授权人) 有效时强制镜像到 AccountableUserID(审计人)；无授权人的自动触发（rule_owner）则保留「accountable 有值、originator 为 NULL」的唯一合法分歧。迁移 190 再以 `NOT VALID` CHECK 兜底，防其他功能绕过 chokepoint 造成 originator=B/accountable=A。

```go
func finalizeAttribution(r Result) Result {
	if r.UserID.Valid {
		r.AccountableUserID = r.UserID
	}
	return r
}
```
[server/internal/attribution/attribution.go:173-178]()
<!-- anchor: func finalizeAttribution(r Result) Result { … 	return r -->

```sql
ALTER TABLE agent_task_queue
    ADD CONSTRAINT agent_task_queue_accountable_matches_originator
    CHECK (
        originator_source IS NULL
        OR originator_user_id IS NULL
```
[server/migrations/190_agent_task_attribution_invariant_check.up.sql:51-58]()
<!-- anchor: ALTER TABLE agent_task_queue …     NOT VALID; -->

**关键设计点 3：fail-closed 是工作区可选的合规开关。** 迁移 188 给 workspace 加 `attribution_fail_closed` 布尔：默认 FALSE 时归因瀑布降级到 owner_fallback（总有负责人），TRUE 时「无法归因到精确人类就拒绝入队」——合规工作区宁可挡单也不跑未归因任务。

```sql
ALTER TABLE workspace
    ADD COLUMN attribution_fail_closed BOOLEAN NOT NULL DEFAULT FALSE;
```
[server/migrations/188_workspace_attribution_fail_closed.up.sql:13-14]()
<!-- anchor: ALTER TABLE workspace …     ADD COLUMN attribution_fail_closed BOOLEAN NOT NULL DEFAULT FALSE; -->

**关键设计点 4：审计不止归因，还有「slug 保留字」这类部署即校验的审计迁移。** 043 在迁移里直接 `RAISE EXCEPTION` 列出冲突 slug 阻止部署——URL 重构后遗留 slug 若撞保留字会破坏路由，宁可阻断发版也不带病上线；保留字清单与 Go/TS 侧同步维护。

```sql
DO $$
DECLARE
  conflict_count INT;
```
[server/migrations/043_audit_reserved_slugs.up.sql:13-35]()
<!-- anchor: DECLARE … END $$; -->

**与核心链路的关系**：归因在每次任务入队边界被 stamp 到 `agent_task_queue.originator_*` 列，审计/成本/授权三线分离——授权读 originator，审计读 accountable，成本与合规读 attribution coverage 指标。注：attribution 数据列 schema 与 data-model 模块存在交集，本组聚焦包级设计，schema 演进从略。

## 组八：middleware / logger / metrics / featureflag / profiling —— 横切基础设施

**一句话定位**：请求横切面（认证、限流、client 元数据）、结构化日志、Prometheus 指标、特征开关、pprof——全部以「环境变量驱动、默认 fail-safe/no-op」的方式注入。

**关键设计点 1：Auth middleware 是四分支凭证裁决 + actor 来源权威化的唯一入口。** 顺序固定：cookie 认证需 CSRF（非安全方法）；`mat_`（task）→ 写 `X-User/Actor/Agent/Task/Workspace-ID` 五头；`mcn_`（cloud）→ Fleet 校验 + owner 存在性双保险；`mul_`（PAT）→ sha256 查库 + 短 TTL 缓存（`last_used_at` 每 TTL 窗口最多刷一次）；JWT → HMAC 验签。PAT 缓存 TTL 会按剩余寿命收缩，避免快过期 token 靠缓存续命。

```go
	if fromCookie && !auth.ValidateCSRF(r) {
		slog.Debug("auth: CSRF validation failed", "path", r.URL.Path)
		http.Error(w, `{"error":"CSRF validation failed"}`, http.StatusForbidden)
```
[server/internal/middleware/auth.go:71-75]()
<!-- anchor: 		if fromCookie && !auth.ValidateCSRF(r) { … 			http.Error(w, `{"error":"CSRF validation failed"}`, http.StatusForbidden) -->

**关键设计点 2：限流用 Lua 脚本保证 INCR+EXPIRE 原子，防「卡住的键变永久封禁」。** Redis 固定窗口按 `path:ip` 计键；`rdb == nil` 时中间件是 no-op（fail-open）；`X-Forwarded-For` 只在直连 IP 命中可信代理 CIDR 时才取链中最右非可信 IP——默认不信任 XFF，防代理后伪造 IP 绕过。

```go
var rateLimitScript = redis.NewScript(`
local count = redis.call('INCR', KEYS[1])
if count == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[1])
end
return count
`)
```
[server/internal/middleware/ratelimit.go:19-25]()
<!-- anchor: var rateLimitScript = redis.NewScript( … return count -->

**关键设计点 3：feature flag 是「配置驱动发布开关」，且已退役 flag 要持续以 enabled 发布以兼容旧桌面客户端。** `internal/featureflags/keys.go` 定义发布 flag（billing_workspace_subscriptions、composio_mcp_apps、plugins_v1 均默认关）+ 一组 compat flag：`agents_agent_builder` 等不再作为发布开关，但持续向旧客户端发布 true——因为旧 desktop 客户端对该键 fail-closed（默认 false），不发布就会把它们的功能关掉。

```go
	// BillingWorkspaceSubscriptions gates the workspace-scoped entitlement,
	// Stripe Checkout, seat reconcile, and Billing Portal proxy surface. It is
	// deliberately off by default so the main repository can ship before the
	// managed cloud enables its matching billing.subscriptions capability.
	BillingWorkspaceSubscriptions = "billing_workspace_subscriptions"
```
[server/internal/featureflags/keys.go:9-41]()
<!-- anchor: const ( … 	resourceLabelsCompat = "settings_resource_labels" -->

**关键设计点 4：日志遵循 slog + tint，TTY 检测决定是否染色；profiling 独立于业务端口。** `logger.Init` 按 stderr 是否终端决定 ANSI 色（重定向到文件时保持干净）；daemon 用 `NewWriterLoggerDefault` 让注入式 logger 与全局 slog 落入同一 managed 文件。pprof 独立监听 `127.0.0.1:6060`，读写超时留给调用者选择（CPU profile/trace 可长跑）。

```go
func Init() {
	level := parseLevel(os.Getenv("LOG_LEVEL"))
	handler := tint.NewHandler(os.Stderr, &tint.Options{
```
[server/internal/logger/logger.go:30-38]()
<!-- anchor: func Init() { … 	slog.SetDefault(slog.New(handler)) -->

**与核心链路的关系**：这些中间件包在每次请求进入时执行认证/限流/埋点，是核心链路的守卫层与观测层；feature flag 决定计费/Composio/Plugin 三类功能面是否对外可见。

## 组九：scheduler —— DB-backed 执行记录调度器（基础设施底座）

**一句话定位**：把 `sys_cron_executions` 表变成所有内部周期任务的「分布式锁 + 审计日志」，每个实例 tick 同一批 job，靠表上唯一键保证只有一个实例拿到租约，输家静默 no-op（MUL-2957 RFC）。

**关键设计点 1：JobSpec 把「失败处理、stale 租约窃取、重试策略、追赶行为」全部声明化。** Cadence 网格化 plan_time + ScheduleDelay 让「刚刚到达的数据」不被迟到处理漏掉；`CatchUpMode` 区分 latest-only（适合 handler 自带 watermark，如 task_usage rollup）与 every-plan（每个 plan 桶有独立业务含义）；`PlansForScope` 钩子让任意 cron 表达式的 autopilot 触发可以完全接管 plan 生成。

```go
type JobSpec struct {
	// Name is the canonical job identifier. Use snake_case ASCII.
	Name string

	// Cadence is the plan bucket size (e.g. 5 * time.Minute). The
	// scheduler floors `db_now - ScheduleDelay` to a multiple of
```
[server/internal/scheduler/spec.go:102-192]()
<!-- anchor: type JobSpec struct { … 	Handler Handler -->

**关键设计点 2：runClaimed 把 handler panic 当作错误处理，防「panic 被吞、却写 SUCCESS 审计行」。** deferred recover 用命名返回值回传 `ErrHandlerPanic`，`classifyError` 把超时/取消/丢租约/panic 映射成短错误码写进审计行，且每个 job 无论是否可重入都先 `markStaleAsFailed` 关掉遗弃的 RUNNING 租约（保 gauge 真实 + 保留重试路径）。

```go
		defer func() {
			if r := recover(); r != nil {
				log.Error("scheduler handler panic", "panic", r)
				retErr = fmt.Errorf("%w: %v", ErrHandlerPanic, r)
			}
		}()
```
[server/internal/scheduler/manager.go:359-365]()
<!-- anchor: 		defer func() { … 		return job.Handler(runCtx, HandlerInput{ -->

**关键设计点 3：租约靠 heartbeat 续期，`lease_token` 保护一切终态写入。** 长 handler 必须周期心跳（Heartbeat 注入 HandlerInput），heartbeat 用脱离请求的 background context 防慢取消丢续期；终态 SUCCESS/FAILED 都带 `lease_token` 条件写——若租约已被窃取则写失败并警告，绝不覆盖新持有者的结果。

```go
	if err := finishSuccess(context.Background(), m.pool, c.ID, c.LeaseToken,
		dbTime, dur, res); err != nil {
		if errors.Is(err, ErrLeaseLost) {
			log.Warn("scheduler: terminal SUCCESS ignored, lease was stolen",
```
[server/internal/scheduler/manager.go:416-423]()
<!-- anchor: 	if err := finishSuccess(context.Background(), m.pool, c.ID, c.LeaseToken, … 		log.Error("scheduler: write terminal SUCCESS", -->

**与核心链路的关系**：scheduler 是 rollup（组四）、autopilot cron 触发、plugin hook 等后台执行的统一底座——核心任务编排的「定时侧面」全挂在它上面。注：若 task-orchestration 模块已深挖本包，本组仅补了 panic 处理与租约设计两个盲区。

## 组十：util / selfexec / runtimeapps —— 工具库

**一句话定位**：无业务语义的通用件：文本/JSON 消毒、进程自定位、任务级外部应用挂载元数据。

**关键设计点 1：文本消毒防「NUL/坏 UTF-8 把 INSERT 打回滚、任务卡 running」。** `SanitizeTextForPostgres` 是 GH #7098 事故的疗愈：`strings.ToValidUTF8` 不覆盖 NUL（0x00 是合法 UTF-8 的 U+0000），必须显式 ReplaceAll；`SanitizeJSONForPostgres` 递归清洗整棵 JSON 树（键也洗），深度上限 32 防对抗性载荷栈溢出。

```go
func SanitizeTextForPostgres(s string) string {
	// Overwhelmingly the common case: leave ordinary text completely alone.
	if !strings.ContainsRune(s, 0) && utf8.ValidString(s) {
		return s
	}
```
[server/internal/util/text.go:79-85]()
<!-- anchor: func SanitizeTextForPostgres(s string) string { … 	return strings.ReplaceAll(strings.ToValidUTF8(s, "\uFFFD"), "\x00", "") -->

**关键设计点 2：selfexec 双保险定位可执行文件，适配启动环境元数据缺失的场景。** 优先 `os.Executable`，失败则回退 argv[0] + `exec.LookPath` + Abs + 常规文件校验，并 `errors.Join` 保留两条失败路径。

```go
func Resolve() (string, error) {
	return resolveWith(os.Executable, os.Args)
}
```
[server/internal/selfexec/selfexec.go:16-18]()
<!-- anchor: func Resolve() (string, error) { … 	return resolveWith(os.Executable, os.Args) -->

**关键设计点 3：ConnectedApp 是「非秘密的任务级挂载元数据」，与秘密 MCP overlay 分离。** 用于 daemon 给 agent 做简报时列出本次 run 实际挂载了哪些外部 app 能力；`DisplayNameForToolkitSlug` 在入队路径零目录调用生成展示名。

```go
type ConnectedApp struct {
	Provider    string `json:"provider"`
	ServerName  string `json:"server_name"`
	ToolkitSlug string `json:"toolkit_slug"`
	ToolkitName string `json:"toolkit_name,omitempty"`
}
```
[server/internal/runtimeapps/connected_app.go:10-15]()
<!-- anchor: type ConnectedApp struct { … 	ToolkitName string `json:"toolkit_name,omitempty"` -->

**与核心链路的关系**：消毒件是 agent 产出内容写入 DB 前的最后防线（task/comment/issue 写入共用）；ConnectedApp 是 task 认领→agent 简报链路的输入。

## 组十一：handler 顶层概览 —— 路由面归纳

**一句话定位**：`server/internal/handler/` 约 110 个非测试 .go 文件承载全部 HTTP 路由面，按领域切片组织（agent*/issue*/chat*/autopilot*/runtime*/plugin*/workspace*/daemon* 等）。

**关键设计点 1：路由面与支撑模块一一对应，同一 handler 文件可同时依赖多个支撑包。** `handler.go` 的 `Config` 集中了部署开关（AllowSignup、DisableWorkspaceCreation、VCSIntegrationEnabled、TrustedProxies、CloudURL 等），并注入了 entitlement/seatcapacity/issuestatus/featureflag 等全部支撑依赖——路由层是各支撑模块的汇聚点。

```go
type Config struct {
	AllowSignup         bool
	AllowedEmails       []string
	AllowedEmailDomains []string
```
[server/internal/handler/handler.go:67-98]()
<!-- anchor: type Config struct { … 	PublicURL string -->

**关键设计点 2：支撑能力以「handler 辅助方法」形式被复用而非复制。** 例如 `seatCapacityEnabled()` 在 invitation.go、share_link.go、workspace.go、workspace_revoke.go、seat_capacity.go 多处被调用（约 14 处），构成座位协议在成员生命周期里的统一开关；share_link 加入流程内嵌了「事务内 Claim 链接 → 容量 token → 加入 → 过渡 confirm」的原子序列。

```go
	var capacityToken uuid.UUID
	if h.seatCapacityEnabled() {
		capacityToken, err = h.beginShareJoinCapacity(r.Context(), uuid.UUID(linkPreview.WorkspaceID.Bytes), uuid.UUID(linkPreview.ID.Bytes), uuid.UUID(user.ID.Bytes))
```
[server/internal/handler/share_link.go:303-310]()
<!-- anchor: 	var capacityToken uuid.UUID … 			return -->

**关键设计点 3：分享链接的「public 预览 → 认证后加入」两段式与容量协议衔接。** `GetShareLinkInfo` 无认证只给工作区名/slug/邀请人/角色（刻意排除 workspace_id、邀请码、邮箱）；`JoinByShareLink` 认证后先取容量 token，再事务内原子 Claim 链接（条件 UPDATE 重新验证 active/未过期/未超 max_uses 并自增），并发加入不可能超卖 max_uses。

```go
	link, err := qtx.ClaimShareLinkByID(r.Context(), linkPreview.ID)
	if err != nil {
		writeError(w, http.StatusNotFound, "share link not found or expired")
```
[server/internal/handler/share_link.go:325-329]()
<!-- anchor: 	link, err := qtx.ClaimShareLinkByID(r.Context(), linkPreview.ID) … 		return -->

**与核心链路的关系**：handler 层不做业务决策，只做「解析→鉴权→调 service/支撑包→序列化」，支撑模块的可用性直接决定哪些路由能开。

## 次要支撑层的整体模式总结

这批模块共享几条可辨识的设计哲学，可作为 README「支撑能力」章节的素材：

1. **Fail-closed 与 fail-open 各安其位**：认证/合规边界 fail-closed（未配置的 `mcn_` 前缀直接 401、attribution fail-closed 拒单、JWT 弱密钥拒绝启动）；缓存/遥测/额度边界 fail-open 或降级（entitlement 缓存过期降 observe、analytics 队列满即丢、ratelimit Redis 挂了放行）。判断标准是「失效时损失的是安全还是体验」。
2. **租约化 / 幂等化无处不在**：scheduler 用 `sys_cron_executions` 唯一键 + lease_token 做分布式锁；seat capacity 用 operation_token + lease 做 outbox；quota 用 idempotency_key 做 CAS；task_usage rollup 用「全量重算非 delta」保证重放幂等。
3. **DB 是唯一的对账方**：管道与后台任务尽量把「锁、watermark、审计、幂等」下沉到 SQL 层（advisory lock 4242/4244/4246、watermark 表、triggers 入脏队列、CHECK 不变量），Go 侧只做壳与编排——代价是要警惕 session 级锁泄漏这类 SQL 层事故（272 是现成教训）。
4. **迁移纪律即运维契约**：无外键/级联、索引一律 CONCURRENTLY 单语句、NOT VALID→VALIDATE 两步大表加约束、回填前向修复（079）、部署即审计（043）、fast metadata-only DDL（188/213）——每个迁移注释都在解释「为什么这么做、不这么做会怎样」。
5. **凭证与身份分层**：长期凭证（JWT/PAT/daemon token）+ 运行时最小权限凭证（`mat_` task token）+ 委托云校验凭证（`mcn_`），`X-Actor-Source` 服务端权威标记使「agent 动作」与「人类动作」在权限、审计、计费三面可区分。
6. **面向 Agent/CLI 的契约设计**：`DeriveKey` 产出 ASCII 机器 handle、错误文案教 agent 怎么改（`allow_duplicate=true`）、CLI 以 `mat_` 前缀区分 401 语义、`X-Client-Capabilities` 做能力协商——对外接口显式考虑 LLM/CLI 消费方。

