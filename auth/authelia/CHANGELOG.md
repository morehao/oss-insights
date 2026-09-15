---
type: 变更记录
title: Authelia 解读变更记录
description: Authelia 项目解读的跨版本变更汇总，按版本倒序，记录每个解读基线的依据 ref 与主要变更。
tags: [authelia, changelog, release-notes]
---

# 变更记录

> 本解读对 Authelia 的跨版本变更汇总；按版本倒序。上游有官方 release notes / 迁移文档的，参照它并结合代码变动生成；没有的部分基于代码变动（git diff / commit 区间）生成。

## [v4.39.26]

依据：`0b553eb` → `v4.39.26`@`f6fb0a3`（323 提交；参照上游 `docs/content/configuration/storage/migrations.md` 与 `docs/content/blog/release-notes-4.39/`，其余基于代码变动）

上游官方 changelog 在此区间主要是逐次 patch 发布（`release: v4.39.21` ~ `v4.39.26`），schema 迁移表给出了最权威的版本对应关系，以下按主题归纳。

### Schema 与数据演进

- 加密 AAD 由「表+列」（schema 25，**该版本从未正式发布**）收紧为「**表+列+行**」（schema 26）。因需全库重加密，`V0026.StorageAADRowScoped.up.sql` 在三后端均为空文件，实际重加密由 Go 侧 `migrationSpecialUp26` 完成；WebAuthn 凭据的 AAD 额外绑定 RPID（`GetIssuer`）。三后端迁移数 25 → 29。
- 新增 V0027 / V0028：`oauth2_refresh_token_session.access_signature` 把 refresh token 会话绑定到其配套 access token 的签名，列长按 RFC9068 `at+jwt` 形态修正为 `VARCHAR(768)`。
- 新增 V0029：全部 `oauth2_*` 会话表增加 `requested_resource` / `granted_resource` 两列，`oauth2_par_context` 与 `oauth2_consent_preconfiguration` 各增加 resource 列。

### 新增能力与行为变化

- **OAuth 2.0 Resource Indicators（RFC8707）**：resource 从 audience 中分离，贯穿「授权请求 → consent → 会话 → token 校验」整条链；预配置 consent 的匹配指纹扩展为 scope+audience+resource 三者全等。
- **Consent 会话与授权请求强绑定**：`OAuth2ConsentSession.MatchesRequester` 逐项比对 client_id / scope / audience，PAR 场景特判 `request_uri`，常规场景比对 `nonce` 与 `state`。
- **LDAP 连接池**：`PooledLDAPClientFactory` 在凭据与配置默认值不一致时自动降级为不池化；修复「池化客户端被错误跳过」的选择逻辑。
- **file 用户库**：热重载时经 `LoadAliases` 重建 email/CI 别名映射，修复失效别名残留导致的误导性错误。
- **端点限流配置化**：`server.endpoints.rate_limits.*` 支持逐端点 `enable` 与多档 `buckets`；新增 `second_factor_password` 档（passkey 回落口令补 2FA 的路径）；令牌桶过期清理由中央 `GarbageCollector` 统一调度；配置 key 总数 271 → 272。
- **内嵌静态资源预压缩**：启动时一次性生成 brotli/gzip 两种表示并各自携带 `ETag`、响应加 `Vary: Accept-Encoding`；首屏线上字节 503,058 → 168,337（`br`），整棵静态树 1,750,748 → 695,593。

### 接口与实现重构

- `RegisterOpenIDConnectRoutes` 移除 `ctx context.Context` 参数。
- `AuthzContext` 新增 `GetXForwardedHost` / `GetXForwardedURI`（缺头时回退请求行），新增 `authorization.NewObjectMethodURLOrSchemeHostPath`；Legacy 反代策略改用之。
- 授权域名匹配：`{user}` / `{group}` 弃用 token 统一翻译为命名分组正则，移除 `len(domain) > len(m.Name)` 的长度比较启发式——此前解读把它记为风险，本版该实现已消除。
- 前端从 Material UI 迁移到 Tailwind CSS + shadcn/Base UI；consent portal 重设计。
- `internal/oidc` 非测试代码约 6762 → 7407 行；新增 `internal/oidc/conformance` 子包（非测试约 353 行）对接 OpenID Foundation 官方认证套件。

### 配置校验与默认值修正

- JWK `direct` 算法的密钥长度检查方向修正（此前写反，导致该算法实际不可用），现要求密钥至少 32 字节。
- Active Directory 默认属性集移除 `website`（`wWWHomePage`），需要者须显式配置。

### 文档一致性同步

- 能力清单、技术栈表、核心模块导读表、比较表述与 README 概览基线行已同步更新（基线 `v4.39.26`@`f6fb0a3`）。
- 全量代码引用已按新基线重定位校验（`verify-references.mjs` 全过）；Mermaid 图经 `check-mermaid.mjs` 校验。
