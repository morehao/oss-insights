# Wiki.js 系统架构设计详解

> **关联文档:** [Wiki.js 主解读](./README.md) · [数据库设计](./db-design.md)

## 1. 架构总览

Wiki.js 采用**前后端分离的单体 SPA 架构**，前端使用 Vue 2 SPA，后端使用 Node.js + Express + Apollo GraphQL，通过 GraphQL 协议通信。首屏渲染由 Express 通过 Pug 模板引擎完成（SSR），后续页面交互由 SPA 通过 Apollo Client 处理。

```
┌──────────────────────────────────────────────────────────────────┐
│                        浏览器 (Vue SPA)                           │
│    Apollo Client (BatchHttpLink + WebSocketLink)                 │
│    Vue 2 + Vuetify 2 + Vuex + vue-router                        │
└──────────────────────┬───────────────────────────────────────────┘
                       │  HTTP POST /graphql  |  WS /graphql-subscriptions
                       │  JWT Cookie / Authorization Header
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                  Express 4 + Apollo Server 2                       │
│                                                                   │
│  中间件链: Security → CORS → Session → Passport → GraphQL → SEO  │
│  REST 路由: /login, /logout, /a/* (Admin), /e/* (Editor), /*     │
└──────────────────────┬───────────────────────────────────────────┘
                       │
┌──────────────────────┼───────────────────────────────────────────┐
│  Core Modules       │            │                               │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ Kernel  |  Config  |  DB (Knex)  |  Auth (Passport+JWT)  │    │
│  │ Servers |  Cache   |  Scheduler  |  Extensions  |  Mail  │    │
│  │ Logger  |  Telemetry | Localization | Let's Encrypt       │    │
│  └──────────────────────────────────────────────────────────┘    │
│                      │                                            │
│  ┌───────────────────┴────────────────────────────────────────┐   │
│  │  Models (Objection.js ORM) — 22 个数据模型                  │   │
│  │  pages | users | groups | pageHistory | tags | assets ...  │   │
│  └───────────────────┬────────────────────────────────────────┘   │
│                      │                                            │
│  ┌───────────────────┴────────────────────────────────────────┐   │
│  │  Module System — 9 大模块类别, 50+ 插拔模块                 │   │
│  │  Auth(21) | Storage(11) | Search(6) | Editor(7) | Render   │   │
│  │  Logging | Analytics | Comments | Extensions               │   │
│  └────────────────────────────────────────────────────────────┘   │
│                      │                                            │
└──────────────────────┼───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                  Database (PostgreSQL/MySQL/MariaDB/MSSQL/SQLite) │
└──────────────────────────────────────────────────────────────────┘
```

## 2. 核心技术栈

| 层次 | 技术选型 | 版本 | 关键特性 |
|------|----------|------|----------|
| **运行时** | Node.js | >= 20 | ES2020+ |
| **Web 框架** | Express | 4.18 | 中间件链 |
| **API 层** | Apollo Server 2 | 2.x | Schema Directive, Subscriptions |
| **ORM** | Objection.js + Knex | 2.2 / 0.21 | 关联查询、迁移、流式查询 |
| **前端框架** | Vue 2 + Vuetify 2 | 2.6 / 2.3 | 组件系统 |
| **状态管理** | Vuex + vuex-pathify | 3.x | 自动 mutation 生成 |
| **GraphQL 客户端** | Apollo Client 2 | 2.x | BatchHttpLink, Subscriptions |
| **认证** | Passport.js + JWT | RS256 | 21 种策略 |
| **模板引擎** | Pug | 2.x | SSR 首屏渲染 |
| **构建工具** | Webpack 4 + Babel | 4.x | 多入口打包 |
| **测试** | Jest + Cypress | - | 单元 + E2E |
| **日志** | Winston | 3.x | 多 Transport |
| **任务调度** | 内置 Scheduler | - | node-cron 风格 |

## 3. 启动流水线

Wiki.js 的启动分为三个阶段，由 `server/index.js` → `server/core/kernel.js` → `server/master.js` 串联：

```
server/index.js（入口）
  │
  ├── 1. 创建 global.WIKI 全局对象
  │     ├── IS_DEBUG / IS_MASTER / INSTANCE_ID
  │     ├── configSvc（配置服务）
  │     └── kernel（内核管理）
  │
  ├── 2. WIKI.configSvc.init()
  │     └── 加载 config.yml → 合并默认值 → 设置 WIKI.config
  │
  ├── 3. WIKI.logger = Winston 日志初始化
  │
  └── 4. WIKI.kernel.init()
        │
        ├── 阶段一：数据库初始化
        │   └── WIKI.models = db.init()
        │         ├── 创建 Knex 连接池（支持 5 种数据库，重试最多 10 次）
        │         ├── 自动加载 server/models/ 下 22 个模型
        │         ├── 执行数据库迁移（migrateFromBeta → syncSchemas）
        │         └── 如果是 PostgreSQL：启用 pg-pubsub HA 事件监听
        │
        ├── 阶段二：运行时配置加载
        │   ├── 尝试从 DB settings 表加载动态配置
        │   ├── DB 配置覆盖 config.yml
        │   └── 为空则进入安装向导模式
        │
        └── 阶段三：bootMaster() 服务启动
              │
              ├── 安装向导模式 → require('../setup')() → 启动 Setup HTTP Server
              │
              └── 正常启动模式：
                    ├── preBootMaster()
                    │     ├── Telemetry 初始化
                    │     ├── Sideloader（离线语言包/插件加载）
                    │     ├── Cache（node-cache 内存缓存实例）
                    │     ├── Servers（注册 Express 中间件 + GraphQL + 订阅）
                    │     ├── Scheduler（注册定时任务）
                    │     ├── EventEmitters（进程内 + 跨实例事件总线）
                    │     ├── Extensions（加载可选扩展）
                    │     └── Asar（打包资源加载）
                    │
                    └── master() — Express 应用组装
                          ├── 安全中间件（helmet, CORS, rate-limit）
                          ├── 会话中间件（cookie-parser, express-session, KnexSessionStore）
                          ├── Passport 初始化 + JWT 验证中间件
                          ├── body-parser（JSON, urlencoded）
                          ├── Apollo Server applyMiddleware (POST /graphql)
                          ├── 静态资源服务 (_assets/ 7d 缓存)
                          ├── Pug 模板引擎 + i18n 中间件
                          ├── 控制器路由 (auth, upload, ssl, common)
                          ├── 404 / 500 错误处理
                          └── 启动 HTTP/HTTPS 服务器
                                │
                                └── postBootMaster()
                                      ├── 刷新所有模块配置（认证/存储/搜索/编辑器等）
                                      ├── 激活认证策略
                                      ├── 初始化搜索引擎
                                      ├── 初始化存储目标
                                      └── 启动 Scheduler 定时任务
```

### 关键设计决策

| 决策 | 理由 |
|------|------|
| **全局单例 `WIKI`** | 所有模块通过 `global.WIKI` 访问，省去依赖注入框架的复杂度 |
| **双阶段启动** | `master()` 完成后端路由组装，`postBootMaster()` 完成模块后初始化，保证系统先可用再加载插件 |
| **连接重试** | DB 连接最多重试 10 次（间隔 3 秒），适配容器化部署中数据库晚于应用启动的场景 |
| **优雅关闭** | 监聽 `SIGTERM`/`SIGINT`，按序关闭服务器 → 调度器 → DB 连接 |

## 4. 中间件链（执行顺序）

中间件注册在 `server/master.js` 中，按以下顺序执行：

```
1. mw.security          ← Helmet 安全头, XSS 过滤, rate limiting（auth 路由 10 req/min）
2. cors                 ← CORS 限制（默认同源）
3. favicon              ← 图标
4. /_assets             ← 静态资源服务（7 天缓存）
5. ctrl.ssl             ← SSL 重定向（配置了证书时）
6. cookie-parser        ← cookie 解析
7. express-session      ← Knex Session Store 持久化会话
8. passport.initialize  ← Passport 初始化
9. WIKI.auth.authenticate ← JWT 验证 + 自动续期 + 吊销检查
10. body-parser json    ← JSON 请求体解析
11. body-parser urlencoded ← 表单请求体解析
12. Apollo Server       ← GraphQL applyMiddleware (POST /graphql, WS /graphql-subscriptions)
13. mw.seo              ← SEO 中间件（301 斜杠重定向等）
14. view engine         ← Pug 模板引擎初始化
15. localization        ← i18n 中间件（从 URL 或 cookie 设置 locale）
16. ctrl.auth           ← 认证路由 (/login, /logout, /register)
17. ctrl.upload         ← 文件上传路由
18. ctrl.common         ← 通配路由 /*（页面查看、管理后台、编辑器等）
19. 404 handler
20. 500 error handler
```

## 5. 路由设计

Wiki.js 使用 Express 控制器做首屏 SSR，页面交互通过 GraphQL 完成。

| 路由 | 控制器 | 功能 | 渲染方式 |
|------|--------|------|----------|
| `GET /login` | auth | 登录页 | Pug SSR |
| `GET /login/:strategy` | auth | 第三方 OAuth 跳转 | 重定向 |
| `GET /logout` | auth | 登出 | 重定向 |
| `GET /register` | auth | 注册页 | Pug SSR |
| `POST /graphql` | Apollo | GraphQL API | JSON |
| `WS /graphql-subscriptions` | Apollo WS | 实时订阅 | WebSocket |
| `GET /a`, `/a/*` | common | 管理后台 | Pug SSR + Vue SPA |
| `GET /e`, `/e/*` | common | 编辑器 | Pug SSR + Vue SPA |
| `GET /h`, `/h/*` | common | 历史版本 | Pug SSR + Vue SPA |
| `GET /s`, `/s/*` | common | 页面源码 | Pug SSR + Vue SPA |
| `GET /*` | common | 页面查看（核心路由） | Pug SSR + Vue SPA |

### 页面渲染流程（GET /some-page）

```
1. Helmet 安全头 → 2. CORS → 3. Cookie 解析 → 4. Session 恢复
→ 5. Passport JWT 验证（从 Cookie/Bearer 提取，含自动续期）
→ 6. i18n 中间件（locale 检测）
→ 7. common.get('/*')
      │
      ├── pageHelper.parsePath()       ← 解析路径，提取 locale
      ├── WIKI.models.pages.getPage()  ← 缓存→DB 读取页面
      ├── WIKI.auth.checkAccess()      ← 权限校验
      ├── 渲染 sidebar（导航树）
      ├── 渲染 page.pug（Vue SPA 容器 + 服务端注入数据）
      └── 浏览器加载 Vue SPA → Apollo Client 完成后续交互
```

## 6. GraphQL 层设计

### 6.1 Schema 组织

GraphQL schema 定义在 `server/graph/` 目录下：

```
graph/
├── index.js           ← Schema 拼接 + Resolver 合并
├── schemas/           ← 20 个 .graphql 文件（类型定义）
│   ├── common.graphql    ← Query/Mutation/Subscription 骨架 + @auth 指令定义
│   ├── page.graphql      ← 页面 CRUD
│   ├── user.graphql      ← 用户管理
│   ├── group.graphql     ← 用户组（含 PageRule 类型定义）
│   ├── authentication.graphql
│   └── ...
├── resolvers/         ← 19 个 resolver 文件
│   ├── page.js
│   ├── user.js
│   ├── group.js
│   └── ...
├── directives/        ← 自定义 Schema Directive
│   └── auth.js        ← @auth(requires: [...]) 实现
└── scalars/           ← 自定义标量类型
```

### 6.2 关键设计模式

**Schema 拼接：** `common.graphql` 定义骨架，其他 schema 使用 `extend type Query {}` 扩展：

```graphql
# common.graphql
type Query
type Mutation
type Subscription

directive @auth(requires: [String]) on QUERY | FIELD_DEFINITION | ARGUMENT_DEFINITION

# page.graphql
extend type Query {
  pages: PageQuery
}
extend type Mutation {
  pages: PageMutation
}
```

**Resolver 嵌套分组：** 通过空对象实现命名空间：

```javascript
// resolvers/page.js
module.exports = {
  Query: { async pages() { return {} } },     // 命名空间
  PageQuery: {
    async single(obj, args, context) { ... },
    async search(obj, args, context) { ... },
  },
  PageMutation: {
    async create(obj, args, context) { ... },
    async update(obj, args, context) { ... },
  }
}
```

**`@auth` 指令：** 字段级权限拦截，在 resolver 执行前检查 `context.req.user.permissions`：

```javascript
// graph/directives/auth.js
if (!requiredScopes.some(s => context.req.user.permissions.includes(s))) {
  throw new Error('Forbidden')
}
```

**实时订阅：** 通过 `WIKI.GQLEmitter = new PubSub()` 推送日志/事件到管理后台。

## 7. 模块化插件系统

### 7.1 九大模块类别

| 模块类型 | 作用 | 官方数量 | 示例 |
|---------|------|---------|------|
| `authentication` | 认证策略 | 20+ | local, oidc, saml, ldap, github, google |
| `storage` | 存储后端 | 11 | git, disk, s3, azure, dropbox, sftp |
| `search` | 搜索引擎 | 6 | db, elasticsearch, algolia, postgres |
| `rendering` | 渲染引擎 | 10+ | markdown-core, mermaid, katex, plantuml |
| `editor` | 编辑器 | 7 | markdown, ckeditor, code, api, asciidoc |
| `logging` | 日志后端 | 4 | disk, sentry, loggly, papertrail |
| `analytics` | 分析服务 | 4 | google, matomo, plausible, umami |
| `comments` | 评论系统 | 2 | default, disqus |
| `extensions` | 通用扩展 | 3 | nlp, security-headers, sitemap |

### 7.2 模块生命周期

```
postBootMaster()
  ↓
models.xxx.refreshXxxFromDisk()    ← 扫描 modules/xxx/ 目录，同步到 DB
  ↓
models.xxx.initTargets()           ← 动态 require() 启用的模块，调用 init()
  ↓
scheduler.start()                  ← 注册定时任务（如 Git 存储的 PT5M 同步）
```

### 7.3 模块结构

每个模块是一个独立目录，包含：

```
modules/storage/git/
├── definition.yml     ← 模块元数据（名称、配置 schema、支持动作、调度频率）
├── storage.js         ← 实现文件（init/created/updated/deleted/renamed/sync）
└── locales/           ← 本地化文件
```

**`definition.yml` 示例：**

```yaml
key: git
title: 'Git'
description: 'Store page content in a Git repository'
schedule: PT5M
supportedModes:
  - sync
  - push
  - pull
actions:
  - handler: syncUntracked
    label: 'Add Untracked Changes'
  - handler: importAll
    label: 'Import Everything'
config:
  - key: repoUrl
    type: string
    label: 'Remote Repository URL'
    required: true
  # ...
```

### 7.4 鸭子类型约定

所有存储模块通过方法名约定实现多态（无需接口或类继承）：

```javascript
// 统一调用方式（storage.js）
static async pageEvent({ event, page }) {
  for (let target of this.targets) {
    await target.fn[event](page)  // created / updated / deleted / renamed
  }
}
```

## 8. 数据流

### 8.1 完整请求路径

```
浏览器操作（如创建页面）
  │
  ├── Vue 组件 → this.$apollo.mutate({ mutation: CREATE_PAGE, variables })
  │
  ├── Apollo Client BatchHttpLink → POST /graphql（JWT Cookie 自动携带）
  │
  ├── Express 中间件链（安全 → 会话 → 认证 → GraphQL）
  │
  ├── Apollo Server → Schema 解析 → @auth 指令校验 → Resolver
  │
  ├── Resolver → Model 层（Objection.js）→ Knex Query Builder → DB
  │
  └── 返回数据 → JSON → Apollo Client → Vue 响应式更新
```

### 8.2 页面变更后处理链

```
页面更新 → pageModel.updatePage()
  │
  ├── 1. addVersion()            → pageHistory 快照
  ├── 2. 更新 DB pages 记录
  ├── 3. associateTags()          → 更新标签关联
  ├── 4. renderPage()             → 提交渲染 Job（异步）
  ├── 5. rebuildTree()            → 提交重建树 Job（异步）
  ├── 6. searchEngine.updated()   → 更新搜索索引
  ├── 7. storage.pageEvent()      → 同步到所有存储后端
  ├── 8. pageLinks.refresh()      → 更新链接关系
  └── 9. deletePageFromCache()    → 清除缓存 → HA 事件广播
```

### 8.3 HA 集群事件传播

在 PostgreSQL 环境下，通过 `pg-pubsub` + `LISTEN/NOTIFY` 实现跨实例通信：

```
实例 A: 页面更新 → deletePageFromCache
  → WIKI.events.outbound.emit('deletePageFromCache', hash)
  → NOTIFY wiki, '{"event": "deletePageFromCache", "hash": "..."}'

实例 B: LISTEN wiki → 收到事件
  → WIKI.events.inbound.emit('deletePageFromCache', hash)
  → deletePageFromCache(hash) 清除本地缓存
```

## 9. 缓存体系

Wiki.js 采用**三级缓存架构**，优先读缓存，缓存未命中再读数据库：

| 层级 | 实现 | 缓存对象 | TTL | 失效方式 |
|------|------|----------|-----|----------|
| **L1 内存** | node-cache | 导航树、语言列表、分析代码 | 300s | 超时 + 事件 |
| **L2 文件系统** | js-binary .bin | 页面渲染结果、资产数据 | 无限 | 显式删除（页面变更时） |
| **L3 HA 传播** | pg-pubsub | 跨实例 L2 缓存一致性 | - | NOTIFY 广播 |

## 10. 配置体系

双层配置架构：

```
┌─────────────────────────────┐
│   config.yml（磁盘文件）     │  ← 基础设施配置：端口、DB 连接、SSL、日志级别
│   路径: /wiki/config.yml    │
├─────────────────────────────┤
│   settings 表（数据库 KV）   │  ← 运行时配置：站点信息、主题、认证、邮件
│   KEY → JSON VALUE          │
└─────────────────────────────┘
         ↓ 合并（DB 覆盖文件）
┌─────────────────────────────┐
│    WIKI.config 运行时对象     │
└─────────────────────────────┘
```

环境变量可覆盖部分文件配置：`DATABASE_URL`、`DB_PASS_FILE`、`CONFIG_FILE` 等。

## 11. 前端架构

### 11.1 目录结构

```
client/
├── client-app.js          ← Vue SPA 入口（Apollo / Vuex / Vuetify / i18n）
├── client-setup.js        ← 安装向导入口
├── index-app.js / index-setup.js / index-legacy.js ← Webpack 多入口
│
├── components/            ← Vue 组件（80+）
│   ├── admin/             ← 管理后台组件（30+）
│   ├── editor/            ← 编辑器组件
│   ├── common/            ← 公共组件（header, search, loader）
│   └── *.vue              ← 页面组件（Page, Login, Register...）
│
├── store/                 ← Vuex 状态管理
│   ├── page.js            ← 页面状态
│   ├── site.js            ← 站点配置
│   ├── user.js            ← 用户认证
│   ├── editor.js          ← 编辑器状态
│   └── admin.js           ← 管理后台
│
├── graph/                 ← GraphQL 查询/变更 (.gql)
├── themes/default/        ← 默认主题（组件 + SCSS + JS）
└── helpers/lib/scss/      ← 工具库/样式
```

### 11.2 前端状态管理

Vuex store 使用 `vuex-pathify` 自动生成 mutations，减少样板代码：

```javascript
// store/page.js
import { make } from 'vuex-pathify'

export default {
  state: {
    ...make.state({ id: null, title: '', content: '', ... })
  },
  mutations: { ...make.mutations(state) },  // 自动生成
  getters: { ... },
  actions: { ... }
}
```

### 11.3 前端权限控制

前端通过 Vuex store 中的 `user.permissions` 和 `page.effectivePermissions` 控制 UI 可见性：

```vue
<v-btn v-if="hasPermission(['write:pages', 'manage:system'])">编辑</v-btn>
```

前端权限控制仅为 UI 层优化，真正的权限校验始终在服务端 GraphQL `@auth` 指令和权限引擎中执行。

## 12. 高可用设计

| 特性 | 实现 | 适用场景 |
|------|------|----------|
| 无状态认证 | JWT（RS256），令牌包含权限信息 | 任意实例处理请求 |
| 配置同步 | pg-pubsub NOTIFY 广播配置变更 | PostgreSQL 集群 |
| 缓存一致性 | 缓存失效事件跨实例传播 | PostgreSQL 集群 |
| 令牌吊销 | 用户/组级别版本号，存储在内存 | 任意实例 |
| 数据库连接池 | Knex 连接池 + 重试 + SSL | 所有数据库 |
| 会话持久化 | express-session + Knex Session Store | 任意实例 |

非 PostgreSQL 环境下，HA 广播降级为单实例模式，基础功能不受影响。

---

> **GitHub:** [requarks/wiki](https://github.com/requarks/wiki) — `server/core/` | `server/master.js` | `server/graph/`
