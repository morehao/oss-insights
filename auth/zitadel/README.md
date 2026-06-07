# ZITADEL — 企业级 IAM 平台解读

> **GitHub:** [github.com/zitadel/zitadel](https://github.com/zitadel/zitadel) | **网站:** [zitadel.com](https://zitadel.com)

## 第一部分：快速概览

### 这是什么

ZITADEL 是一个开源的企业级身份与访问管理平台（Identity and Access Management），提供完整的认证、授权、多租户和审计能力。它采用 Go + Angular/React 技术栈，API First 设计，支持 OIDC/SAML/SCIM 等标准协议，既可自托管部署也有 SaaS 云服务。

### 解决的痛点

技术团队面临一个共性矛盾：自建认证系统成本高（Session 管理、MFA、OIDC 协议实现、安全合规），而现成方案要么过重（Keycloak 的 Java 栈）、要么封闭（Auth0 商业闭源）、要么职责单一（OAuth2-proxy 仅做代理认证）。ZITADEL 在开源与企业级完整度之间取得平衡——多协议、多租户、Event Sourcing 审计、双架构迁移。

### 适用场景

- 多租户 SaaS 平台的统一身份认证基座
- 企业内部 SSO 与身份联邦（OIDC + SAML + LDAP）
- B2B 场景中每个客户独立 Instance（租户隔离）的认证方案
- 需要完整审计追溯的合规场景
- 需要自定义登录 UI 的品牌化认证体验

### 核心能力清单

| 能力 | 说明 |
|------|------|
| OIDC 完整实现 | Authorization Code + PKCE、Implicit、Device Auth、Refresh Token、Back-Channel Logout |
| SAML 2.0 | 服务提供者（SP）支持，元数据管理 |
| SCIM | 用户与组的跨域身份管理协议 |
| 多种认证因子 | 密码、Passkey（WebAuthn）、TOTP、OTP SMS/Email、Recovery Code、LDAP、IDP 联邦 |
| 多租户隔离 | Instance → Organization → Project 三级层次模型，数据物理隔离 |
| RBAC 权限体系 | 25+ 角色类型，细粒度权限映射，Manager Role 委托管理 |
| Event Sourcing | 所有状态变更通过事件记录，完整审计链，事件表保留历史 |
| 双架构迁移 | 从 Event Sourcing 逐步迁移到关系型系统记录（V2/V3 API 使用新架构） |
| 自定义登录 UI | Next.js 登录应用，支持品牌化定制（颜色、Logo、文案） |
| Actions/Webhook | 生命周期钩子，在认证流关键节点执行自定义脚本 |
| 管理控制台 | Angular Admin Console，可视化管理用户/组织/项目/策略 |
| 部署方式 | Docker Compose / Kubernetes Helm / 云服务 |
| 可观测性 | OpenTelemetry 全链路追踪、Prometheus 指标、ServicePing |
| 多语言 | 内置 i18n 国际化，登录 UI 和管理控制台均支持 |

### 技术栈一览

| 层面 | 技术 |
|------|------|
| 语言 | Go 1.25 |
| HTTP 框架 | go-chi/chi、gorilla/mux |
| RPC 协议 | connectRPC（V2）、gRPC、HTTP/JSON |
| API 定义 | Protobuf（connectRPC + gRPC + REST） |
| 数据库 | PostgreSQL（主）、CockroachDB（部分支持） |
| 事件存储 | 自研 Eventstore（eventstore.events2 表） |
| 配置 | Viper + YAML + 环境变量（ZITADEL_ 前缀） |
| 前端 | Angular（管理控制台）+ Next.js/React（登录 UI） |
| 认证 | go-jose（JWT/JWE）、webauthn（WebAuthn）、crewjam/saml（SAML） |
| 加密 | AES-256-GCM、bcrypt、RSA/SHA-256 |
| 缓存 | Memory / PostgreSQL / Redis（断路器模式） |
| 构建 | Nx + pnpm（monorepo 编排） |
## 第二部分：项目架构

### 整体架构

```
┌──────────────────────────────────────────────────────────────┐
│                        客户端层                                │
│  ┌──────────┐  ┌────────────────┐  ┌──────────────────────┐  │
│  │ CLI/API  │  │ Angular 控制台  │  │ Next.js 登录界面     │  │
│  └────┬─────┘  └───────┬────────┘  └──────────┬───────────┘  │
└───────┼────────────────┼──────────────────────┼──────────────┘
        │                │                      │
        ▼                ▼                      ▼
┌──────────────────────────────────────────────────────────────┐
│                        API 网关层                              │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  connectRPC (V2)  │  gRPC  │  HTTP/JSON (REST)         │  │
│  └───────────────────────┬─────────────────────────────────┘  │
└──────────────────────────┼────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                        命令层 (Command)                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────┐  │
│  │  认证     │  │  管理员  │  │  用户    │  │  系统管理    │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬──────┘  │
└───────┼─────────────┼─────────────┼───────────────┼──────────┘
        │             │             │               │
        ▼             ▼             ▼               ▼
┌──────────────────────────────────────────────────────────────┐
│                        领域层 (Domain)                         │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  聚合 (Aggregate)                                       │  │
│  │  ┌──────┐ ┌──────┐ ┌─────┐ ┌────────┐ ┌──────────────┐ │  │
│  │  │ User │ │ Org  │ │IAM │ │Project│ │Instance     │ │  │
│  │  └──┬───┘ └──┬───┘ └──┬──┘ └───┬────┘ └──────┬───────┘ │  │
│  │     │ PushEvent│        │       │              │         │  │
│  │     ▼          ▼        ▼       ▼              ▼         │  │
│  │  ┌───────────────────────────────────────────────────┐  │  │
│  │  │  Eventstore (events2 表)                          │  │  │
│  │  └───────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

### 单体仓库结构

```
zitadel/
├── apps/
│   ├── login/          # Next.js 登录应用（认证页面、注册、MFA）
│   ├── docs/           # Fumadocs 文档站点
│   └── api/            # API 入口点（Nx 项目目标）
├── console/            # Angular 管理控制台
├── internal/           # 后端核心逻辑
│   ├── api/            # API 层（gRPC/connectRPC 处理器）
│   │   ├── grpc/       # gRPC 服务实现
│   │   ├── oidc/       # OIDC 端点实现
│   │   └── ui/login/   # 登录 UI 后端处理器
│   ├── command/        # 命令处理器（CQRS 写模型）
│   ├── query/          # 查询处理器（CQRS 读模型）
│   ├── domain/         # 领域模型（聚合、实体、值对象）
│   ├── eventstore/     # 事件存储引擎
│   ├── database/       # 数据库迁移 & Flyway
│   └── static/i18n/    # 国际化翻译文件
├── proto/              # Protobuf 定义
│   └── zitadel/        # API proto 文件
├── packages/           # 共享 TypeScript 包
├── e2e/                # 端到端测试（Go）
├── cmd/                # 入口点与配置
│   ├── defaults.yaml   # 默认配置
│   └── zitadel/        # 主程序入口
├── pkg/                # 可导出的公共包
├── docs/               # 开发者文档
├── deploy/             # 部署配置
│   └── compose/        # Docker Compose
├── .github/            # CI/CD（GitHub Actions）
├── go.mod              # Go 模块定义
├── nx.json             # Nx 编排配置
├── pnpm-workspace.yaml # pnpm monorepo
└── tests/
    └── functional-ui/  # Cypress 功能测试
```

### 关键模块职责

| 模块 | 职责 |
|------|------|
| `cmd/zitadel/` | 程序入口：初始化、依赖注入、启动服务 |
| `internal/api/grpc/` | gRPC 和 connectRPC 的 API 服务器实现 |
| `internal/api/oidc/` | OIDC Provider 端点（/authorize、/token、/userinfo 等） |
| `internal/api/ui/login/` | 服务端渲染的登录页面处理器（传统路径） |
| `apps/login/` | 新一代 Next.js 登录 UI（独立部署，现代前端栈） |
| `internal/command/` | CQRS 命令侧：接收指令，验证规则，生成事件 |
| `internal/query/` | CQRS 查询侧：直接从关系表读取数据 |
| `internal/domain/` | 领域模型：聚合根、实体、值对象、领域事件定义 |
| `internal/eventstore/` | 事件存储引擎：写入事件流，支持重放和投影 |
| `console/` | Angular 管理控制台：可视化管理组织/用户/项目 |
| `proto/zitadel/` | 所有 API 的 Protobuf 定义（服务、消息、枚举） |

### CQRS + Event Sourcing

```
写操作（命令）                      读操作（查询）
     │                                  │
     ▼                                  ▼
┌─────────┐                      ┌─────────────┐
│ Command │ ─── 验证 ──► 事件 ──► │ 关系型数据库 │
│ Handler │                      │ (查询投影)    │
└─────────┘                      └─────────────┘
     │                                  ▲
     │  写入事件                         │  直接查询
     ▼                                  │
┌─────────────┐                   ┌───────────┐
│  Eventstore │                   │   Query   │
│  (events2)  │                   │  Handler  │
└─────────────┘                   └───────────┘
```

ZITADEL 正在经历架构迁移：
- **V1 API**：完全基于 Event Sourcing，从事件重放还原状态
- **V2/V3 API**：关系型数据为主记录（system of record），事件仅作审计

这意味着新增的查询处理器直接从 PostgreSQL 关系表读取，不再经历完整的事件重放流程，显著提升查询性能。
## 第三部分：多租户架构

### 三层层次模型

```
System (安装级别)
 └── Instance A (实例/租户)       ← 数据物理隔离
      ├── Organization 1          ← 管理边界
      │    ├── Project Alpha      ← 应用集合
      │    │    ├── App A1 (OIDC)
      │    │    └── App A2 (SAML)
      │    └── Users / Groups / Roles
      ├── Organization 2
      │    └── Project Beta
      └── ...
 └── Instance B                   ← 完全独立的租户
      ├── Organization X
      └── ...
```

### Instance 是什么

在 ZITADEL 中，**Instance（实例）** 不是一般意义上的"例子"，而是一个完整的逻辑租户/分区——一个系统内部的独立系统。它用于：

- **SaaS 多租户隔离**：每个客户拥有独立的 Instance，数据和策略完全隔离
- **环境分离**：开发/测试/生产各自独立 Instance
- **企业多部署单元**：大规模企业中不同业务线各自独立 Instance

### 角色与权限范围

| 角色类型 | 管辖范围 | 说明 |
|----------|----------|------|
| System User | 整个安装 | 管理系统级配置、创建 Instance |
| IAM Owner | 单 Instance | Instance 内最高权限 |
| Org Owner | 单 Organization | 管理组织内的用户和项目 |
| Project Owner | 单 Project | 管理项目中的应用和设置 |
| Manager Role | 可委托 | 细粒度委托管理权限 |

### Console 权限映射（25+ 角色）

ZITADEL 管理控制台定义了 25+ 细粒度角色：
- `IAM_OWNER`、`IAM_ADMIN_IMPERSONATION`、`IAM_END_USER_IMPERSONATOR`
- `ORG_OWNER`、`ORG_USER_MANAGER`、`ORG_OWNER_VIEWER`
- `ORG_PROJECT_PERMISSION_EDITOR`、`ORG_GRANT_ACTIVE_EDITOR`
- `ORG_DOMAIN_DNS_EDITOR` 等

每个角色精确定义了哪些 Console Management API 可以被调用。

### 数据隔离方式

```
┌────────────────────────┐
│  数据库                │
│  ┌──────────────────┐  │
│  │ instance_id = A  │  │  ← 所有表通过 instance_id 隔离
│  └──────────────────┘  │
│  ┌──────────────────┐  │
│  │ instance_id = B  │  │  ← Instance B 无法访问 A 的数据
│  └──────────────────┘  │
└────────────────────────┘
```

所有核心表（`projects`、`orgs`、`users` 等）都带 `instance_id` 字段，查询时强制过滤，确保租户间数据物理隔离。

### 部署拓扑

```
                    负载均衡器
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   ZITADEL-A        ZITADEL-B        ZITADEL-C
        │                │                │
        └────────────────┼────────────────┘
                         │
                   PostgreSQL 集群
                   (主从 + 连接池)
```

多实例 ZITADEL 共享同一个 PostgreSQL 集群，通过 `instance_id` 实现逻辑隔离。对于高性能场景，可按 Instance 分配到不同的数据库 Shard。
## 第四部分：认证体系全景

### 多协议支持

| 协议 | 角色 | 功能 |
|------|------|------|
| OIDC (OpenID Connect) | Provider (OP) | Authorization Code + PKCE、Implicit、Device Auth、Refresh Token、Back-Channel Logout、Client Credentials |
| SAML 2.0 | Service Provider (SP) | 接收外部 IdP 的 SAML Assertion、元数据管理 |
| LDAP | 集成 | 通过 LDAP Identity Provider 连接外部目录服务 |
| SCIM | 支持 | 用户与组的跨域身份管理（System for Cross-domain Identity Management） |

### 认证方式总览

```
┌─────────────────────────┐
│  多因素认证 (MFA)        │
│  ┌─────────────────────┐ │
│  │ Password            │ │  ← 传统密码，强度策略控制
│  │ Passkey (WebAuthn)  │ │  ← 无密码认证，生物/安全密钥
│  │ TOTP                │ │  ← 基于时间的一次性密码
│  │ OTP SMS / Email     │ │  ← 短信/邮件验证码
│  │ Recovery Code       │ │  ← 恢复码
│  │ LDAP                │ │  ← 外部 LDAP IdP
│  │ IDP Federation      │ │  ← 身份提供商联邦（OIDC IdP）
│  └─────────────────────┘ │
└─────────────────────────┘
```

### OIDC 认证流

```
用户 → 应用 (RP)
 │       │
 │       ▼
 │    重定向到 ZITADEL /authorize
 │       │
 │       ▼
 │    ┌──────────────────────────────┐
 │    │  ZITADEL OIDC Provider       │
 │    │                              │
 │    │  1. 验证 client_id / redirect│
 │    │  2. 展示登录界面 (apps/login) │
 │    │  3. 用户认证 (可选 MFA)       │
 │    │  4. 生成授权码                │
 │    │  5. 回调 redirect_uri        │
 │    └──────────────────────────────┘
 │       │
 │       ▼
 │    应用用 code 换 token
 │       │
 │       ▼
 │    ┌──────────────────────────────┐
 │    │  POST /oauth/v2/token        │
 │    │  → access_token (JWT)        │
 │    │  → id_token (JWT)            │
 │    │  → refresh_token             │
 │    └──────────────────────────────┘
 │       │
 │       ▼
 │    应用用 id_token 获取用户信息
 │       │
 │       ▼
 │    ┌──────────────────────────────┐
 │    │  GET /oidc/v1/userinfo       │
 │    │  → sub, name, email, ...     │
 │    └──────────────────────────────┘
```

### Token 类型与安全

| Token | 格式 | 签名算法 | 说明 |
|-------|------|----------|------|
| Access Token | JWT (Opaque) | RS256 | 用于访问 API，可配置不透明模式 |
| ID Token | JWT | RS256 | 包含用户身份声明 |
| Refresh Token | Opaque | - | 用于续期 access_token |
| Service User Token | JWT | RS256 | 机器对机器通信的持久 Token |
| Personal Access Token (PAT) | Opaque | - | 用户个人访问令牌 |

### 登录 UI 定制

ZITADEL 提供两个登录界面路径：

1. **传统路径** (`internal/api/ui/login/`)：Go 模板渲染，集成在 API 进程中
2. **新一代** (`apps/login/`)：Next.js 独立应用，支持动态品牌化

```
┌────────────────────────────────┐
│  apps/login/ (Next.js)         │
│  ┌──────────────────────────┐  │
│  │ 主题系统                  │  │
│  │  - 品牌色                 │  │
│  │  - Logo / Favicon         │  │
│  │  - 自定义 CSS             │  │
│  │  - 多语言文本              │  │
│  │  - 邮件模板               │  │
│  └──────────────────────────┘  │
│  ┌──────────────────────────┐  │
│  │ 页面                     │  │
│  │  - /login                │  │
│  │  - /password-reset       │  │
│  │  - /mfa                  │  │
│  │  - /register             │  │
│  │  - /device               │  │
│  └──────────────────────────┘  │
└────────────────────────────────┘
```

每个 Instance 可以拥有独立的品牌化设置，通过 Instance 级别的策略配置实现。

### Actions（自定义钩子）

```
认证流时间线
─────────────────────────────────────────────────────►
PreAuth  ──► PostAuth  ──► PreCreation  ──► PostCreation

┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Token 补充│   │ 认证后    │   │ 用户创   │   │ 创建后    │
│ 自定义    │   │ 处理逻辑  │   │ 建前验证  │   │ 处理      │
└──────────┘   └──────────┘   └──────────┘   └──────────┘
```

Actions 是在认证流关键节点触发的自定义 JavaScript 函数，支持：
- Token 动态添加自定义 Claims
- 用户创建/更新时的数据校验与变形
- 外部服务 HTTP 调用（Webhook 集成）
- 条件感知认证（基于上下文允许/拒绝认证）
## 第五部分：Event Sourcing 详解

### 设计原理

ZITADEL 的核心写入路径采用 Event Sourcing 模式：

```
命令 → 验证 → 事件 → 持久化
(Cmd) → (Validate) → (Event) → (Eventstore)
```

每次状态变更（创建用户、修改角色、删除项目）不直接修改数据库行，而是追加一个不可变的事件记录。当前状态通过事件重放（Replay）计算得到。

### 事件存储结构

```sql
-- 事件表（events2）
CREATE TABLE eventstore.events2 (
    id SERIAL PRIMARY KEY,
    event_type VARCHAR NOT NULL,         -- 例如 "user.added"
    aggregate_type VARCHAR NOT NULL,      -- 例如 "user"
    aggregate_id VARCHAR NOT NULL,        -- 例如 "123456"
    aggregate_version VARCHAR NOT NULL,   -- 版本号 "v1"
    event_sequence BIGINT NOT NULL,       -- 全局递增序列号
    previous_aggregate_sequence BIGINT,   -- 上一个事件的序列号
    previous_aggregate_type_sequence INT, -- 类型内上一个序列
    creation_date TIMESTAMPTZ NOT NULL,   -- 事件时间
    editor_user TEXT NOT NULL,            -- 操作用户
    editor_service TEXT,                  -- 操作服务
    resource_owner TEXT NOT NULL,         -- 资源所有者（org/instance）
    instance_id TEXT NOT NULL,            -- 实例 ID
    "position" DECIMAL NOT NULL,          -- 事件位置编号
    event_data JSONB,                     -- 事件负载（JSON）
    creator TEXT NOT NULL,                -- 创建者
    -- 唯一约束
    UNIQUE (aggregate_id, aggregate_type, aggregate_version)
);
```

### 事件类型示例

| 聚合 | 事件类型 | 描述 |
|------|----------|------|
| user | `user.added` | 用户创建 |
| user | `user.changed` | 用户资料变更 |
| user | `user.removed` | 用户删除 |
| user | `user.human.mfa.init.skipped` | 跳过 MFA 初始化 |
| user | `user.human.password.changed` | 密码修改 |
| org | `org.added` | 组织创建 |
| org | `org.changed` | 组织信息变更 |
| project | `project.added` | 项目创建 |
| project | `project.application.added` | 应用添加 |
| instance | `instance.added` | 实例创建 |
| instance | `instance.policy.login.added` | 登录策略变更 |

事件的 `event_data` 字段包含变更的完整负载（JSON），例如用户创建事件：

```json
{
  "userName": "alice",
  "firstName": "Alice",
  "lastName": "Smith",
  "email": "alice@example.com",
  "displayName": "Alice S.",
  "preferredLanguage": "zh",
  "password": {
    "secret": {
      "cryptoType": 1,
      "algorithm": "bcrypt",
      "data": "..."
    },
    "changeRequired": false
  }
}
```

### 投影（Projection）

从事件重建可查询状态的过程叫做投影：

```
   events2 表
     │
     ▼
┌──────────────────────────────┐
│  Projection Handler          │
│  - 监听新事件                 │
│  - 更新关系型表               │
│  - 维护物化视图               │
└──────────────────────────────┘
     │
     ▼
  关系型查询表
  (projects, users, orgs, ...)
```

投影处理器持续监听 `events2` 表的新事件，将其转换为关系型表的增删改操作。这个机制使得：
- **写路径**保持 Event Sourcing 的完整审计能力
- **读路径**可以像传统 SQL 一样高效查询

### 架构迁移：V2/V3 的变更

```
V1 架构（纯 Event Sourcing）:
  ┌──────┐    ┌────────┐    ┌──────────┐    ┌────────┐
  │ 命令  │ →  │ 验证   │ →  │ Eventstore│ →  │ 重放    │ → 当前状态
  └──────┘    └────────┘    └──────────┘    └────────┘

V2/V3 架构（混合模式）:
  ┌──────┐    ┌────────┐    ┌──────────┐    ┌──────────┐
  │ 命令  │ →  │ 验证   │ →  │ Eventstore│    │ 关系表    │ ← 系统记录
  └──────┘    └────────┘    │ (仅审计)  │    │ (主记录)  │
                             └──────────┘    └──────────┘
                                    │              │
                                    ▼              ▼
                              ┌──────────────────────┐
                              │ 事件表保留完整历史     │
                              │ 关系表是 system of     │
                              │ record（系统记录）     │
                              └──────────────────────┘
```

新 API 不再通过事件重放确定状态，而是直接从关系表读取。事件流仅保留用于审计目的。
