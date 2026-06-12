# Wiki.js 数据库设计详解

> **关联文档:** [Wiki.js 主解读](./README.md) · [多版本文档机制](./version-management.md) · [文档资源权限管理](./permission-model.md)

## 1. 设计哲学

Wiki.js 的数据库设计遵循以下原则：

| 原则 | 体现 |
|------|------|
| **跨数据库兼容** | 使用 Knex 抽象层，时间字段统一存 ISO 8601 字符串而非原生时间类型，长文本用数据库特性分支处理 |
| **读写分离** | 内容(content)与渲染结果(render)分离存储，渲染通过异步 Job 在后端完成 |
| **缓存优先** | 三层缓存架构（内存+文件系统+HA 事件传播），页面读取性能优先于写入 |
| **物化路径** | pageTree 使用物化路径+祖先缓存加速树形结构查询 |
| **插件配置复用** | 认证/存储/搜索/编辑器等 8 类模块共用 Key-Config 配置表模式 |
| **二进制内容分离** | 资产元数据(assets)与二进制数据(assetData)分离存储 |

## 2. 数据库支持

通过 Knex 查询构建器，支持 5 种数据库后端：

| 数据库 | 连接包 | 推荐场景 |
|--------|--------|----------|
| PostgreSQL | `pg` | 生产环境（推荐，支持 HA pub/sub） |
| MySQL / MariaDB | `mysql2` | 生产环境 |
| MSSQL | `mssql` | Windows 环境 |
| SQLite | `sqlite3` | 开发/个人部署 |

数据库类型检测与适配逻辑在 `server/core/db.js` 中，通过 `WIKI.config.db.type` 分发差异化的 SQL/配置处理。

## 3. 核心表结构

### 3.1 页面体系

#### pages — 核心页面表

```sql
pages (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  path            VARCHAR   NOT NULL,           -- URL 路径，如 "guide/getting-started"
  hash            VARCHAR   NOT NULL,           -- SHA1(locale|path|privateNS)，用作缓存文件名
  title           VARCHAR   NOT NULL,           -- 页面标题
  description     VARCHAR,                       -- 页面描述（搜索摘要用）
  isPrivate       BOOLEAN   NOT NULL DEFAULT 0,  -- 私密页面标记
  isPublished     BOOLEAN   NOT NULL DEFAULT 0,  -- 发布状态
  privateNS       VARCHAR,                       -- 私密命名空间（多租户隔离）
  publishStartDate VARCHAR,                      -- 定时发布开始（ISO 8601）
  publishEndDate   VARCHAR,                      -- 定时发布结束（ISO 8601）
  content         TEXT/LONGTEXT,                 -- 原始内容（markdown/html/asciidoc）
  render          TEXT/LONGTEXT,                 -- 渲染后的 HTML
  toc             JSON,                          -- 目录结构（h1-h6 嵌套 JSON）
  contentType     VARCHAR   NOT NULL,            -- 内容类型：markdown / html / asciidoc
  extra           JSON,                          -- 自定义 CSS/JS 脚本
  editorKey       VARCHAR,                       -- FK → editors.key
  localeCode      VARCHAR(5),                    -- FK → locales.code
  authorId        INTEGER,                       -- FK → users.id（最后编辑者）
  creatorId       INTEGER,                       -- FK → users.id（创建者）
  createdAt       VARCHAR   NOT NULL,
  updatedAt       VARCHAR   NOT NULL
)
```

**关键设计点：**
- `path` + `localeCode` 联合决定页面唯一性（无显式 UNIQUE 索引，通过代码保证）
- `hash` 通过 `SHA1(locale|path|privateNS)` 生成，用作 `.bin` 缓存文件名
- `content` 与 `render` 分离，前者存原始编辑内容，后者存渲染引擎输出的 HTML
- `toc` 字段用 JSON 存储解析后的目录结构（h1-h6 嵌套数组）
- `extra` 字段存储页面自定义的 CSS/JS，受 `write:styles` / `write:scripts` 权限保护
- 所有时间字段存 ISO 8601 字符串，避免跨数据库的时间类型兼容问题

#### pageHistory — 版本历史表

```sql
pageHistory (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  pageId          INTEGER,                       -- FK → pages.id
  path            VARCHAR   NOT NULL,
  hash            VARCHAR   NOT NULL,
  title           VARCHAR   NOT NULL,
  description     VARCHAR,
  isPrivate       BOOLEAN   NOT NULL DEFAULT 0,
  isPublished     BOOLEAN   NOT NULL DEFAULT 0,
  publishStartDate VARCHAR,
  publishEndDate   VARCHAR,
  action          VARCHAR   DEFAULT 'updated',   -- updated / moved / deleted（initial 由 getHistory 推导）
  content         TEXT/LONGTEXT,                 -- 历史版本完整内容
  contentType     VARCHAR   NOT NULL,
  extra           JSON,
  editorKey       VARCHAR,
  localeCode      VARCHAR(5),
  authorId        INTEGER,
  versionDate     VARCHAR   NOT NULL,            -- 版本标记时间（与 createdAt 独立）
  createdAt       VARCHAR   NOT NULL
)
```

**版本管理策略：**
- 每次 update / move / delete 操作前，**将当前页面完整快照**写入 pageHistory
- `versionDate` 用于标记版本时间线，`createdAt` 记录写入时间
- 支持按 ISO 8601 Duration（如 `P30D`）批量清理旧版本

#### pageTree — 物化路径导航树

```sql
pageTree (
  id            INTEGER PRIMARY KEY,             -- 手动赋值的自增 ID
  path          VARCHAR   NOT NULL,
  depth         INTEGER   UNSIGNED NOT NULL,     -- 树深度
  title         VARCHAR   NOT NULL,
  isPrivate     BOOLEAN   NOT NULL DEFAULT 0,
  isFolder      BOOLEAN   NOT NULL DEFAULT 0,    -- 目录节点 or 叶子节点
  privateNS     VARCHAR,
  parent        INTEGER   UNSIGNED,              -- FK → pageTree.id（自引用）
  pageId        INTEGER   UNSIGNED,              -- FK → pages.id（仅叶子节点有值）
  localeCode    VARCHAR(5),
  ancestors     JSON                             -- 祖先节点 ID 数组（加速权限查询）
)
```

**树结构设计：**
- 使用**物化路径 + 父引用**混合模式
- `rebuild-tree` Job 扫描所有 pages，按 `/` 分割 path 递归构建层级
- 中间目录节点 (`isFolder=true`) 的 `pageId` 为 null，仅作为导航容器
- `ancestors` JSON 字段预计算祖先链，加速页面级权限的祖先路径匹配

#### pageLinks — 页面间链接跟踪

```sql
pageLinks (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  pageId      INTEGER,                           -- FK → pages.id（源页面）
  path        VARCHAR   NOT NULL,                -- 目标页面路径（链接到的路径）
  localeCode  VARCHAR(5) NOT NULL,               -- 目标页面语言
  pageExists  BOOLEAN   NOT NULL DEFAULT 0       -- 目标页面是否存在（断链检测）
)

INDEX: (path, localeCode)
```

**断链检测：** 页面创建/更新/删除时扫描 content，提取所有 wiki 链接，维护 pageLinks 表，标记 `pageExists` 字段。呈现时对有问题的链接加高亮提示。

### 3.2 用户与权限体系

#### users — 用户表

```sql
users (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  email           VARCHAR   NOT NULL,
  name            VARCHAR   NOT NULL,
  providerKey     VARCHAR   NOT NULL DEFAULT 'local', -- FK → authentication.key
  providerId      VARCHAR,                           -- 第三方登录的 provider 用户 ID
  password        VARCHAR,                           -- bcrypt 哈希（rounds=12）
  tfaIsActive     BOOLEAN   NOT NULL DEFAULT 0,
  tfaSecret       VARCHAR,                           -- TOTP 密钥
  jobTitle        VARCHAR   DEFAULT '',
  location        VARCHAR   DEFAULT '',
  pictureUrl      VARCHAR,
  timezone        VARCHAR   NOT NULL DEFAULT 'America/New_York',
  dateFormat      VARCHAR   NOT NULL DEFAULT '',
  appearance      VARCHAR   NOT NULL DEFAULT '',
  localeCode      VARCHAR(5) NOT NULL DEFAULT 'en',
  defaultEditor   VARCHAR   NOT NULL DEFAULT 'markdown',
  isSystem        BOOLEAN   NOT NULL DEFAULT 0,      -- 系统保留账户
  isActive        BOOLEAN   NOT NULL DEFAULT 0,
  isVerified      BOOLEAN   NOT NULL DEFAULT 0,
  mustChangePwd   BOOLEAN   NOT NULL DEFAULT 0,
  lastLoginAt     VARCHAR,
  createdAt       VARCHAR   NOT NULL,
  updatedAt       VARCHAR   NOT NULL,
  UNIQUE(providerKey, email)
)
```

**关键设计点：**
- `UNIQUE(providerKey, email)` — 同一认证策略下邮箱唯一
- 密码使用 bcrypt，rounds=12
- `providerKey` 支持 local + 20+ 第三方 OAuth/OIDC/SAML 策略
- 内置用户：ID=1 Administrator（系统管理员）, ID=2 Guest（匿名访客）

#### groups — 用户组表

```sql
groups (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  name            VARCHAR   NOT NULL,
  permissions     JSON      NOT NULL,               -- 全局权限数组 ["read:pages", "write:pages"]
  pageRules       JSON      NOT NULL,               -- 页面级访问规则数组
  isSystem        BOOLEAN   NOT NULL DEFAULT 0,
  redirectOnLogin VARCHAR   NOT NULL DEFAULT '/',
  createdAt       VARCHAR   NOT NULL,
  updatedAt       VARCHAR   NOT NULL
)
```

- `permissions` — JSON 数组，存储 `action:resource` 格式的权限字符串
- `pageRules` — JSON 数组，每条规则含 `{ deny, match, path, roles, locales }`
- 内置组：ID=1 Administrators（`manage:system`）, ID=2 Guests（`read:pages`）

#### userGroups — 用户-组关联表

```
userGroups (id, userId FK→users.id, groupId FK→groups.id)
```

用户权限 = 所属所有组 `permissions` 的并集 + `pageRules` 的交集。

### 3.3 资产管理

#### assets — 资产元数据表

```sql
assets (
  id        INTEGER PRIMARY KEY AUTOINCREMENT,
  filename  VARCHAR   NOT NULL,
  hash      VARCHAR   NOT NULL,                   -- 内容哈希（缓存文件名）
  ext       VARCHAR   NOT NULL,                   -- 文件扩展名
  kind      ENUM('binary','image') NOT NULL DEFAULT 'binary',
  mime      VARCHAR   NOT NULL DEFAULT 'application/octet-stream',
  fileSize  INTEGER   UNSIGNED,
  metadata  JSON,                                 -- 图片宽高、exif 等
  folderId  INTEGER   UNSIGNED,                   -- FK → assetFolders.id
  authorId  INTEGER   UNSIGNED,                   -- FK → users.id
  createdAt VARCHAR   NOT NULL,
  updatedAt VARCHAR   NOT NULL
)
```

#### assetData — 资产二进制数据表

```sql
assetData (
  id    INTEGER PRIMARY KEY,                      -- FK → assets.id（一对一）
  data  LONGBLOB/BINARY NOT NULL                  -- 文件原始二进制
)
```

**存储策略：** 元数据与二进制分离 -> 文件系统缓存 `cache/{hash}.dat` 优先 -> 数据库兜底。

#### assetFolders — 资产文件夹（自引用树）

```sql
assetFolders (
  id        INTEGER PRIMARY KEY AUTOINCREMENT,
  name      VARCHAR NOT NULL,
  slug      VARCHAR NOT NULL,
  parentId  INTEGER UNSIGNED                      -- FK → assetFolders.id（自引用）
)
```

递归查询使用 CTE（PostgreSQL/MySQL 8+/SQLite 3.8+）：
```sql
WITH RECURSIVE ancestors AS (
  SELECT id, name, slug, parentId FROM assetFolders WHERE id = ?
  UNION ALL
  SELECT a.id, a.name, a.slug, a.parentId FROM assetFolders AS a
  JOIN ancestors ON ancestors.parentId = a.id
)
SELECT * FROM ancestors
```

### 3.4 模块配置表（Key-Config 模式）

以下 8 张表使用完全相同的设计模式：

| 表名 | 主键 | 关键字段 | 说明 |
|------|------|----------|------|
| `authentication` | `key` (VARCHAR) | `isEnabled`, `config` (JSON), `title`, `strategyLink` | 认证策略 |
| `editors` | `key` (VARCHAR) | `isEnabled`, `config` (JSON), `title`, `icon` | 编辑器 |
| `renderers` | `key` (VARCHAR) | `isEnabled`, `config` (JSON), `title`, `icon` | 渲染器 |
| `searchEngines` | `key` (VARCHAR) | `isEnabled`, `config` (JSON), `title` | 搜索引擎 |
| `analytics` | `key` (VARCHAR) | `isEnabled`, `config` (JSON), `title` | 分析服务 |
| `storage` | `key` (VARCHAR) | `isEnabled`, `config` (JSON), `title`, `mode`, `schedule` | 存储目标 |
| `commentProviders` | `key` (VARCHAR) | `isEnabled`, `config` (JSON), `title` | 评论提供者 |
| `loggers` | `key` (VARCHAR) | `isEnabled`, `config` (JSON), `title` | 日志记录器 |

```sql
-- 典型结构（以 authentication 为例）
authentication (
  key           VARCHAR(50) PRIMARY KEY,          -- 模块标识符，如 "local", "oidc"
  isEnabled     BOOLEAN     NOT NULL DEFAULT 0,
  config        JSON        NOT NULL,             -- 模块配置对象（运行时通过管理 UI 编辑）
  title         VARCHAR(100) NOT NULL,            -- 显示名称
  strategyLink  VARCHAR,                           -- 认证策略特有的链接配置
  icon          VARCHAR,                           -- 前端图标
  order         INTEGER     NOT NULL DEFAULT 0,
  selfRegistration BOOLEAN NOT NULL DEFAULT 0,
  domainWhitelist VARCHAR,
  guestLogin    BOOLEAN     NOT NULL DEFAULT 0,
  autoEnrollGroups JSON,                           -- 自动分配用户到哪些组
  createdAt     VARCHAR     NOT NULL,
  updatedAt     VARCHAR     NOT NULL
)
```

**设计优势：** 无需为每个模块类型创建定制表。新增模块只需在 `modules/` 下创建目录和 `definition.yml`，启动时自动注册到对应的配置表中。

### 3.5 其他配置表

#### settings — KV 存储的系统设置

```sql
settings (
  key   VARCHAR PRIMARY KEY,
  value JSON NOT NULL
)
```

存储站点标题、主题、SEO、邮件等所有运行时配置。启动时覆盖 `config.yml` 的对应值。

#### navigation — 导航配置

```sql
navigation (
  key   VARCHAR PRIMARY KEY,                      -- 如 "default", "sidebar"
  config JSON NOT NULL                            -- 导航树完整配置（嵌套 JSON）
)
```

#### locales — 语言配置

```sql
locales (
  code          VARCHAR(5) PRIMARY KEY,            -- "en", "zh-CN", "ja" 等
  isEnabled     BOOLEAN NOT NULL DEFAULT 0,
  isRTL         BOOLEAN NOT NULL DEFAULT 0,        -- 从右到左语言（如阿拉伯语）
  name          VARCHAR(100) NOT NULL,             -- "English"
  nativeName    VARCHAR(100) NOT NULL,             -- "English"
  config        JSON,
  createdAt     VARCHAR NOT NULL,
  updatedAt     VARCHAR NOT NULL
)
```

## 4. 模型关联关系（Objection.js）

```
pages
  ├── tags:        ManyToManyRelation → pageTags → tags（多对多）
  ├── links:       HasManyRelation → pageLinks（一对多，源页面）
  ├── author:      BelongsToOneRelation → users (authorId)
  ├── creator:     BelongsToOneRelation → users (creatorId)
  └── locale:      BelongsToOneRelation → locales (localeCode)

pageHistory
  ├── tags:        ManyToManyRelation → pageHistoryTags → tags
  ├── page:        BelongsToOneRelation → pages (pageId)
  ├── author:      BelongsToOneRelation → users (authorId)
  ├── editor:      BelongsToOneRelation → editors (editorKey)
  └── locale:      BelongsToOneRelation → locales (localeCode)

users
  ├── groups:      ManyToManyRelation → userGroups → groups
  └── locale:      BelongsToOneRelation → locales (localeCode)

groups
  └── users:       ManyToManyRelation → userGroups → users

assets
  ├── author:      BelongsToOneRelation → users (authorId)
  └── folder:      BelongsToOneRelation → assetFolders (folderId)

assetFolders
  └── parent:      BelongsToOneRelation → assetFolders（自引用）

comments
  ├── author:      BelongsToOneRelation → users (authorId)
  └── page:        BelongsToOneRelation → pages (pageId)

pageLinks
  └── page:        BelongsToOneRelation → pages (pageId)

userKeys
  └── user:        BelongsToOneRelation → users (userId)
```

## 5. 查询模式

### 5.1 典型查询

| 场景 | 查询模式 | 说明 |
|------|----------|------|
| 页面查询 | `withGraphJoined('author.creator.tags')` | 关联查询，Objection 生成 JOIN |
| 分页历史 | `orderBy('versionDate', 'desc').page(offset, size)` | Knex 内置分页 |
| 子查询更新 | `whereIn('id', builder => builder.select().from('pageLinks'))` | 关联子查询 |
| 直接更新 | `knex('users').where('id', id).update({ lastLoginAt })` | 绕开 ORM hook |
| 树重建 | `knex.table('pageTree').truncate()` | 全量重建 |
| 流式读取 | `knex.select().from('pages').where({}).stream()` | 大数据量流式处理 |

### 5.2 索引分析

从迁移文件 `2.0.0.js` 中，显式创建的索引极少：

```javascript
table.index(['path', 'localeCode'])  // pageLinks 表
```

**缺乏显式索引的高频查询字段：**

| 查询模式 | 表 | 建议索引 |
|----------|-----|---------|
| 按 path + locale 查页面 | pages | `(path, localeCode)` |
| 按 pageId 查历史 | pageHistory | `(pageId)` |
| 按 userId 查 userKeys | userKeys | `(userId)` |
| 按 tag 查标签 | tags | `(tag)` |
| 按 hash 查资产 | assets | `(hash)` |
| 按 token 查 API Key | apiKeys | `(token)` |

## 6. 迁移机制

### 迁移文件

迁移文件位于 `server/db/migrations/`，按版本号组织：

```
migrations/
├── 2.0.0.js      ← 初始 Schema（创建所有表）
├── 2.1.5.js      ← 增量迁移 1
└── ...
```

迁移使用 Knex.js 迁移 API，每迁移前检查当前数据库版本，确保幂等执行：

```javascript
// 迁移结构
exports.up = async (knex) => {
  // 创建表/修改列/添加索引
}

exports.down = async (knex) => {
  // 回滚（可选）
}
```

### Beta 版本迁移

`db.js` 中的 `migrateFromBeta()` 方法处理从 beta 版本到正式版的 Schema 变更，兼容早期用户数据。

## 7. 高可用事件传播

在 PostgreSQL 环境下，通过 `pg-pubsub` 实现跨实例的缓存失效事件：

```javascript
// db.js - subscribeToNotifications
// LISTEN wiki 通道
// 收到 reloadConfig / flushCache / deletePageFromCache 等事件
// → WIKI.events.inbound.emit() 本地传播
```

非 PostgreSQL 环境下，此功能自动禁用，不会影响单实例运行。

---

> **GitHub:** [requarks/wiki](https://github.com/requarks/wiki) — `server/models/` | `server/db/migrations/`
