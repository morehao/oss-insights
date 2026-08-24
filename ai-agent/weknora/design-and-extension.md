# 设计与扩展性 — WeKnora 深入解析

> 所属项目：[WeKnora](./README.md) · 关注面：模块化可替换 / DI / resource_urls / CLI / 演进

## 根因

WeKnora 的定位是可二次开发的"企业知识管理基础设施"，因此**哪都可以换**是它的设计精髓——模型、向量库、存储后端三层面全部接口化，外加一个面向 Agent 的 CLI 与可覆盖的开发接口。本节聚焦这套可扩展性骨架与 v0.7.2 的 `resource_urls` 文件直链设计。

## 三层面可替换架构

**层面 1 — LLM/Embedding/Rerank 可替换**：所有模型经统一接口接入，新增厂商只需实现 `chat.Chat` 接口（目前 26 家，含 OpenAI 兼容、ollama、Anthropic、火山、智谱等）。Embedding 有专属实现或走 OpenAI 兼容默认。

**层面 2 — 向量数据库可替换**：统一 `RetrieveEngine` 接口 + 注册表模式（详见 rag-engine）。目前可部署独立引擎 7 个 + pgvector/sqlite env store + Neo4j 图谱。

**层面 3 — 存储后端可替换**：文件存储接口抽象，支持 8 种后端——本地/MinIO/COS/TOS/S3/OSS/KS3/OBS。v0.7.0 起支持一个工作区注册多个存储实例并绑定不同知识库，工作区级默认值 + CRUD/连通性测试 API + 凭据脱敏。

## 依赖注入

用 `uber-go/dig` 管理 200+ 依赖提供者，启动按依赖自动解析：无依赖的 LLM/Embedding 先建 → 依赖服务的 Agent/检索/数据源后建 → 路由注册（依赖所有服务）。新增服务只需加提供者 + 注册，启动流程零改动。

## v0.7.2：`resource_urls=public` 文件直链

### 问题

正常情况下聊天回答、引用、检索里的文件和图片是内部 `resource://<handle>` 引用，第三方应用想拿到字节必须**再发一次带凭证的 `/files` 代理请求**。对"外部应用直接渲染"的场景（把回答里的图片直接 load 进 `<img>`），这很不友好。

### 解法

v0.7.2 引入 `RESOURCE_URL_MODE`/`?resource_urls=public`，让响应直接返回可加载的 `http(s)://.../r/<token>` 直链。核心在新增的 `internal/storageurl` 包——它**不是替换 resource://，而是 opt-in 的响应改写层**：默认仍 `handle`（`ModeHandle`），只有显式请求 public 时才把引用翻译成带时间窗的签名 URL。

两种触发方式由 `ResolveMode` 合并：**显式 query 优先于部署默认**，非法 query 值作为错误返回（让集成方发现拼写错误而非静默拿 handle），默认未设时安全回归 handle（[internal/storageurl/mode.go:15-42]()）。

<!-- anchor: // Mode selects how stored files are referenced in an API response. … var ErrPublicModeForbidden = errors.New( -->

**不可绕过的两条安全边界**（`ResolveMode` 强制）：
- **匿名 embed 渠道**被 `WithForcedHandleMode` 钉死在 handle——embed 访客仅凭 channel 的 session handle 授权，必须继续走 channel 作用域的 `/embed/.../files` 代理，不能拿到可分享的免凭证 URL（[internal/storageurl/mode.go:53-59]()）。

<!-- anchor: func WithForcedHandleMode(ctx context.Context) context.Context { … func HandleModeForced(ctx context.Context) bool { -->

- **KB-restricted API key** 请求 public 返回 403（`ErrPublicModeForbidden`）——这种 key 已被拒绝 `/files` 代理，若再给匿名文件 URL 等于从"chunk 文本"后门扩权到"文件字节"（[internal/storageurl/mode.go:135-148]()）。

<!-- anchor: func ResolveMode(ctx context.Context, queryValue string) (Mode, error) { -->

### 每资源复用 grant，让只读端点不写库

`resource_urls` 最精妙的点在于**每资源复用一条存活 grant 而非每次请求插一行**。`CreateAccessGrant` 优先 `reuseOrCreateDerivedGrant`：token 由 HMAC-SHA256 派生、plaintext 不落库只存 hash，但授权仍完全靠 `resource_access_grants` 行（存在且未过期才放行）；只在首批/过期时插入（[internal/application/service/resource.go:166-187]()）。

<!-- anchor: func (s *resourceCatalog) CreateAccessGrant( … if token != "" { -->

注释直言：若不复用，渲染一个答案/回读一条消息历史会对**每图每请求插一行，把读端点变成写重型**。配合 `Rewriter.ref()` 的 memo（同一资源在同一请求跨多个 SSE 分块只解析一次），双层保证低成本。

### 流式分块的正确性

`storageurl/stream.go` 用 `incompleteRefSuffixRe` + `incompleteMarkdownImageSuffixRe` 检测分块边界处被"切一半"的引用或未闭合的图片 URL，`HoldbackCutoff()` 把不完整尾巴暂留到下一 chunk 补全，`FlushAll` 在流中断时释放——避免客户端收到坏链接（与 IM 的 Holdback 同理）。

## CLI（weknora v0.10）

采用 `gh` 风格的 `<noun> <verb>` 命令树，包含 `api`/`auth`/`chat`/`config`/`doc`/`doctor`/`kb`/`search`/`session`/`agent`/`model`/`chunk`/`mcp`/`skills`/`profile`/`link`/`unlink`/`schema`/`exit-codes` 等命令组。亮点：

- **Agent 优先**的 `session ask`/`session resume`/`session stop`/`session tool-approval`。
- `mcp serve` 导出一个只读 stdio MCP 服务器（含 `chat`/`kb_list`/`search_chunks` 等工具），让 Claude Code/Cursor 直接驱动 WeKnora。
- 项目级绑定 `weknora link`（写 `.weknora/project.yaml`）。
- 稳定 JSON 信封、`--jq`/`--format` 字段选择、NDJSON/裸 JSON 输出。
- 多层上下文认证（OS keyring + 0600 文件兜底），支持 API-key 与 JWT 两种模式。

## 演进脉络（v0.6 → v0.7.2）

从 v0.6 到 v0.7.2，WeKnora 从"RAG 产品"走向"企业知识操作系统"：

- **广度扩张**：细粒度 principal 模型与 scoped API Key、运行时队列治理、Embed 挂件、官方文档站（`website-docs/`）、可变连接器（GitLab/IMA/Feishu Drive）、chunk/wiki 修订历史与文件夹树。
- **持续做减法**：移除 Neo4j episodic 记忆、统一 Wiki 操作历史、Langfuse 从自研迁到 OTel 标准、收敛可观测后端——主动降低部署与维护负担。

## 设计决策要义

- **Why 三层面接口化**：企业私有化部署里"模型/向量库/存储"都是可替换的经营决策（成本、合规、已有资产），接口化让替换不侵入核心逻辑，这是二次开发平台的立身之本。
- **Why resource_urls 是 opt-in 改写层而非替换 handle**：直接替换会打破 `/files` 代理的鉴权模型；opt-in 让安全模式与便利模式共存，且用强制边界（匿名 embed、受限 API key）封死降级风险。
- **Why 派生 token + 行校验 + memo 三层**：派生 token 免落库明文、行校验保授权、memo 去重，三者缺一都会在"读端点写库 or 安全性 or 重复签发"之间失衡——这是"便利"与"安全"与"成本"三者的工程平衡点。
