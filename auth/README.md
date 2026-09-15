# 认证授权（Auth）

本目录收录认证授权（Authentication & Authorization）领域的开源项目解读。

| 项目 | 语言 | 说明 | 协议 | 仓库 |
|------|------|------|------|------|
| [Authelia](authelia/) | Go | 反向代理伴生认证/授权 SSO 网关，四种反代集成、细粒度访问控制规则、多因子（WebAuthn/TOTP/Duo）、OpenID Certified OIDC、HA（remote DB + Redis） | Apache-2.0 | [GitHub](https://github.com/authelia/authelia) |
| [Tinyauth](tinyauth/) | Go | 最小认证授权服务器，为反向代理提供 ForwardAuth | AGPL-3.0 | [GitHub](https://github.com/tinyauthapp/tinyauth) |
| [ZITADEL](zitadel/) | Go | 企业级 IAM 平台，CQRS + Event Sourcing 架构，OIDC/SAML/SCIM 完整支持，Hosted Login V2、Actions 编程扩展、多租户 Instance 隔离 | AGPL-3.0 | [GitHub](https://github.com/zitadel/zitadel) |
| [TOPIAM](topiam/) | Java | 企业级 IAM/IDaaS 平台（CE），实际协议为 OIDC/JWT/表单代填，身份源限钉钉/飞书——SAML2/CAS/LDAP/MFA/验证码模块已在 1.0.0 前后移除，与 README 宣称不符 | AGPL-3.0 | [GitHub](https://github.com/topiam/eiam) |
| [MaxKey](maxkey/) | Java | 企业级 IAM-IDaas 单点登录系统，支持 OAuth2/OIDC/SAML2/CAS/JWT 协议，12+ 种认证方式 | Apache-2.0 | [GitHub](https://github.com/dromara/MaxKey) |
| [Keycloak](keycloak/) | Java | IAM 领域标准参考实现，CNCF 孵化，OIDC/SAML2/OAuth2/SCIM/SSF 完整协议支持，可配置认证流程引擎 | Apache-2.0 | [GitHub](https://github.com/keycloak/keycloak) |

## 横向对比

- [`capability-matrix.md`](capability-matrix.md) — 6 家 × 50 项能力横评：用 7 态标记区分「定位外 / 需扩展 / 真缺失」，每格证据可回溯到各项目解读；含选型指引
