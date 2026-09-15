---
type: 变更记录
title: Keycloak 解读变更记录
description: Keycloak 项目解读的跨版本变更汇总，按版本倒序，记录每个解读基线的依据 ref 与主要变更内容。
tags: [keycloak, changelog, release-notes]
---

# 变更记录

> 本解读对 Keycloak 的跨版本变更汇总；按版本倒序。上游在 GitHub Releases 与 `docs/documentation/release_notes/` 维护逐版本条目，本文件只记录「解读内容」自身的基线变化与结论修订，不重复上游发布说明。

## [26.7.3]

依据：`26.7.3`@`6d238b6558`

本次为**全量重写**，替换并移除了此前的 `README.md`（基线 v26.1）、`login-logout-flow.md` 与 `resource-permission-design.md` 三篇旧文档。旧文使用「第 N 部分」式编号标题、无 frontmatter，且源码级引用仅 8 条——`auth/capability-matrix.md` 已把「Keycloak 解读文档的源码级引用很薄」列为待办。新文档集改为语义式标题、统一补齐 frontmatter，并为每条论断附上经 `verify-references.mjs` 程序重定位校验的 `文件:行号` 引用。

文档集（`auth/keycloak/`）：

- `README.md` —— 总览：根因、项目规模、技术栈、子文档索引、核心模块导读表、整体架构、启动与请求链路、设计亮点、问题、同类对比、解读类型说明、阅读路线。
- `provider-spi.md` —— 内核：Provider/SPI 扩展体系与启动装配（一切皆 Provider 的落地与代价）。
- `authentication-flow.md` —— 可配置认证流程引擎：执行树、三态 Requirement、挑战-续跑状态机。
- `oidc-protocol.md` —— OIDC/OAuth2 协议层：授权码全链路、令牌颁发与生命周期、SSO 会话与登出。
- `data-model.md` —— 数据模型与存储：JPA 实体与 Adapter、realm 行级隔离、Liquibase 演进与滚动升级约束、缓存读写路径。
- `authorization-services.md` —— 细粒度授权：Resource/Scope/Policy/Permission 四元模型、决策合并、UMA 与 permission ticket、FGAP。
- `identity-brokering.md` —— 身份代理与用户联合：brokering 全链路、账号链接边界、LDAP 联合与存储委托链。
- `deployment.md` —— 部署与运行时：Quarkus 分发、构建期/运行期配置分层、Feature 开关、HA 与 Operator。
- `protocols-and-management.md` —— 外围协议与管理面：SAML/SCIM/SSF/AuthZEN/OID4VC、Admin REST API 与 Console、密钥与事件体系。

### 本次对 kb 既有论断的修正

- **realm 隔离层级**：`auth/zitadel/README.md` 的同类对比表把 Keycloak 记为「Schema 级（realm 独立 schema）」。按源码，`USER_ENTITY` 等全局共享同一套表，以 `REALM_ID` 列 + 复合唯一约束做**行级隔离**（`model/jpa/src/main/java/org/keycloak/models/jpa/entities/UserEntity.java:61-63,95`），仓库内不存在按 realm 建 schema 的代码。该行已在本次同步更正为「单库共享 schema + `REALM_ID` 行级隔离」。
- **内置口令哈希**：复核确认无 BCrypt 实现，注册的 `PasswordHashProviderFactory` 只有 PBKDF2 三档 + Argon2（`services/src/main/resources/META-INF/services/org.keycloak.credential.hash.PasswordHashProviderFactory`、`crypto/default/src/main/resources/META-INF/services/org.keycloak.credential.hash.PasswordHashProviderFactory`），`auth/capability-matrix.md` 的修正成立。
- **Refresh Token 轮换默认关闭**：复核确认 `RealmEntity.revokeRefreshToken` 是无默认值的原生 `boolean`（默认 `false`），轮换逻辑由 `realm.isRevokeRefreshToken()` 把守（`model/jpa/src/main/java/org/keycloak/models/jpa/entities/RealmEntity.java:107-108`、`services/src/main/java/org/keycloak/protocol/oidc/TokenManager.java:1212,1233`），`auth/capability-matrix.md` 的判定成立。
- **SPI 规模口径**：capability-matrix 记 Keycloak「176 个 SPI」。本次在 26.7.3 基线按 11 个生产模块的 `META-INF/services/org.keycloak.provider.Spi` 实测为 **160 条声明、去重后 159 个不同 SPI 类型**（另有 2 个仅供测试的声明文件不计入），差异来自基线不同与计数口径，已在 `provider-spi.md` 中按本次实测口径重述。
- **Refresh Token 的形态**：常见叙述（含旧版 kb 与本次解读的初稿）称 refresh token 是「不透明串」。按源码它是**签名 JWT**（`core/src/main/java/org/keycloak/representations/RefreshToken.java` 中 `RefreshToken extends AccessToken`，且带 `prov` 字段）。它必须回源，是因为其 `sid` 只是指向会话的索引，会话是否活跃只有存储能答（`services/src/main/java/org/keycloak/protocol/oidc/TokenManager.java` 的 `validateToken`）。已在 `README.md` 与 `oidc-protocol.md` 中更正。
- **`model/storage*` 的职责**：该模块**不是**「新 MAP 存储 API」，而是用户存储联邦的公共契约/实现与缓存抽象（`org.keycloak.storage`、`org.keycloak.models.cache`）。MAP 存储在 26.7.3 已不存在，在册数据存储实现的 provider id 为 `legacy`。已在 `README.md` 与 `data-model.md` 中更正。
- **Admin REST 资源不在 `rest/` 模块**：`rest/` 在 26.7.3 只剩 `admin-v2`（Admin API v2）与 `admin-ui-ext` 两个子模块；45 个传统 Admin REST 资源类位于 `services/src/main/java/org/keycloak/services/resources/admin/`。已在 `README.md` 的模块职责映射中更正。
- **policy-enforcer 不在本仓库**：`authz/` 只有 `client`（HTTP 客户端库）与 `policy`（内置策略实现）；执行侧强制（Policy Enforcer）自 21.1.0 起剥离为独立制品 `keycloak-policy-enforcer`，`adapters/` 亦已在 19.0.0 整体 EOL。已在 `README.md` 与 `authorization-services.md` 中更正。
- **外围协议的成熟度分档**：`Profile.Feature` 中 `SCIM_API` 为 PREVIEW，而 `SSF`、`AUTHZEN`、`OID4VC_VCI` 系列与新一代 `CLIENT_ADMIN_API_V2` 均为 EXPERIMENTAL。README 不再笼统称「完整协议支持」，改为按档位标注。

### 本次解读的校验口径

- 全部 9 篇文档（`README.md` 与 8 篇关注面）共 **895 条引用**经 `verify-references.mjs` 以「首末行双向锚点重定位」方式逐条校验，结果 `WEAK=0 MISMATCH=0 UNVERIFIED=0 TRAVERSAL=0`；全部 Mermaid 图经 `check-mermaid.mjs` 校验通过且 `[MERMAID-WARN]` 为 0。
- 另有两处引用指向 `quarkus/container/Dockerfile`。该文件无扩展名，而校验脚本的引用正则要求 `\.\w+` 形式的路径，故这两处无法被脚本捕获，已由人工逐行核对（`Dockerfile:22/34/36/40` 与锚点一致，且该文件内 `kc.sh build` 出现次数为 0）。
- 本次解读**不设「覆盖率」指标**：文档以关注面为组织单位，边界由小节标题界定。起草阶段用于对齐进度的覆盖率统计属过程产物，未进入正式文档；无法用源码证实或证伪的论断一律删除，不做「可能/大概」式的对冲保留（各篇在重写时据此删除了少量经验性陈述与不可复现的统计数字）。
