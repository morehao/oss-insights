# Wiki.js 文档资源权限管理详解

> **关联文档:** [Wiki.js 主解读](./README.md) · [多版本文档机制](./version-management.md)

## 1. 三层权限架构

Wiki.js 的权限体系分为三个层次，每一层拦截粒度不同：

```mermaid
graph TB
    subgraph "第一层：Schema 层"
        AuthDirective[GraphQL @auth 指令]
        Desc1["检查 JWT permissions 数组<br/>'用户是否有 read:pages 权限？'"]
    end

    subgraph "第二层：逻辑层"
        CheckAccess[checkAccess 权限引擎]
        Desc2["综合判断：<br/>1. manage:system 绕过<br/>2. 全局权限匹配<br/>3. PageRule 页面级规则"]
    end

    subgraph "第三层：执行层"
        ModelCRUD[Model CRUD 方法]
        Desc3["数据操作前二次校验<br/>context 验证（路径/标签/locale）"]
    end

    Request[请求] --> AuthDirective
    AuthDirective -->|通过| CheckAccess
    AuthDirective -->|拒绝| Deny1[❌ Unauthorized]
    CheckAccess -->|通过| ModelCRUD
    CheckAccess -->|拒绝| Deny2[❌ Forbidden]
    ModelCRUD -->|通过| Response[✅ 返回数据]
```

### 第一层：GraphQL @auth 指令

定义在 `server/graph/directives/auth.js`，作为 GraphQL 的 Schema Directive：

```javascript
// server/graph/directives/auth.js
// @auth(requires: ["read:pages", "manage:system"])
// 检查 context.req.user.permissions 是否包含任一所需权限
// 无用户 -> Unauthorized
// 无匹配 -> Forbidden
```

在 schema 中使用：

```graphql
# server/graph/schemas/page.graphql
type Query {
    single(id: Int): Page @auth(requires: ["manage:pages", "delete:pages"])
    history(id: Int!): [PageVersion] @auth(requires: ["manage:system", "read:history"])
}
```

### 第二层：checkAccess 权限引擎

`server/core/auth.js:221` — 这是权限决策的核心函数：

```javascript
// server/core/auth.js:221
static checkAccess(user, permissions, page) {
    // 1. manage:system 绕过
    if (user.permissions.includes('manage:system')) return true

    // 2. 全局权限检查：用户是否拥有任一所需权限
    const hasGlobal = permissions.some(p => user.permissions.includes(p))
    if (!hasGlobal) return false

    // 3. 页面级规则检查（仅当提供 page 对象时）
    if (page) {
        return this._evaluatePageRules(user, permissions, page)
    }
    return true
}
```

### 第三层：Model CRUD

在 `server/models/pages.js` 的数据操作方法中，进一步校验操作上下文：

```javascript
// server/models/pages.js:56 (createPage)
if (!WIKI.auth.checkAccess(user, ['write:pages', 'manage:pages', 'manage:system'], { path })) {
    throw new Error('访问被拒绝')
}
// 还要校验目标路径是否冲突、contentType 是否合法等
```

## 2. 权限清单

Wiki.js 定义了 20+ 个权限字符串，按功能域分组：

| 权限 | 作用于 | 说明 |
|------|--------|------|
| `manage:system` | 全局 | **超管** — 绕过所有权限检查 |
| `read:pages` | 页面 | 查看页面内容 |
| `write:pages` | 页面 | 创建/编辑页面 |
| `manage:pages` | 页面 | 移动/重命名页面 |
| `delete:pages` | 页面 | 删除页面 |
| `read:source` | 页面 | 查看页面源码 |
| `write:styles` | 页面 | 注入自定义 CSS |
| `write:scripts` | 页面 | 注入自定义 JS |
| `read:history` | 版本 | 查看页面历史版本 |
| `read:comments` | 评论 | 查看评论 |
| `write:comments` | 评论 | 发表评论 |
| `manage:comments` | 评论 | 编辑/删除任何评论 |
| `read:assets` | 资源 | 查看/下载文件资源 |
| `write:assets` | 资源 | 上传文件资源 |
| `manage:assets` | 资源 | 重命名/删除文件资源 |
| `write:users` | 用户 | 创建用户 |
| `manage:users` | 用户 | 编辑/删除/激活用户 |
| `write:groups` | 组 | 创建/编辑用户组 |
| `manage:groups` | 组 | 管理高级组（含 admin 权限的组） |
| `manage:navigation` | 导航 | 管理侧边栏导航 |
| `manage:theme` | 主题 | 管理主题样式 |
| `manage:api` | API | 管理 API 密钥及开关 |

## 3. 用户与组模型

### 数据关系

```mermaid
graph TB
    User[users 表] --> UserGroups[userGroups 关联表]
    UserGroups --> Group[groups 表]
    Group --> Permissions[permissions: [String]]
    Group --> PageRules[pageRules: [PageRule]]
```

### 内置用户

| ID | 用户 | 说明 |
|----|------|------|
| 1 | **Root** | 系统管理员，`permissions: ['manage:system']`，不可删除 |
| 2 | **Guest** | 匿名用户，权限来自 "guest" 组，可配置 |

### 权限继承

用户的最终权限 = 所属所有组的 permissions 并集 + PageRule 交集。JWT 令牌中包含聚合后的 `permissions` 数组和 `groups` 数组：

```javascript
// server/models/users.js:225
static async refreshToken(user) {
    const groups = await user.getGroups()
    const permissions = [
        ...new Set(groups.flatMap(g => g.permissions))
    ]
    // 生成 RS256 签名的 JWT
}
```

## 4. PageRule 规则引擎

PageRule 是 Wiki.js 权限体系的精髓，允许在**页面级别**精确控制访问。

### PageRule 数据结构

```graphql
# server/graph/schemas/group.graphql
type PageRule {
    id: Int
    deny: Boolean         # true=拒绝, false=允许
    match: PageRuleMatch  # 匹配类型
    path: String          # 匹配模式
    roles: [String]       # 此规则作用于哪些权限
    locales: [String]     # 可选语言过滤
}

enum PageRuleMatch {
    START   # 路径前缀匹配
    END     # 路径后缀匹配
    EXACT   # 精确路径匹配
    REGEX   # 正则匹配
    TAG     # 标签匹配
}
```

### 五种匹配类型

| 类型 | 示例 | 匹配范围 |
|------|------|---------|
| **START** | `path: "/engineering"` | 匹配 `/engineering/*` 所有子页面 |
| **END** | `path: "/draft"` | 匹配所有以 `/draft` 结尾的路径 |
| **EXACT** | `path: "/admin/settings"` | 仅匹配该精确路径 |
| **REGEX** | `path: "^/draft-\d+"` | 正则匹配路径（启用前安全校验） |
| **TAG** | `path: "confidential"` | 匹配所有打了 `confidential` 标签的页面 |

### 规则解析流程

```mermaid
graph TB
    Start[获取用户所有组的 pageRules] --> Merge[合并规则列表]
    Merge --> Specificity[按特异性排序]
    Specificity --> Evaluate[逐条评估]

    Evaluate --> Match{路径/标签匹配?}
    Match -->|匹配| Action[执行 deny/allow]
    Match -->|不匹配| Next[下一条]
    Next --> Evaluate

    Action --> Deny{deny 规则?}
    Deny -->|是| Block[❌ 拒绝访问]
    Deny -->|否| Pass[✅ 允许访问]

    subgraph "特异性优先级"
        P1["1. END（最高）"]
        P2["2. REGEX"]
        P3["3. EXACT"]
        P4["4. TAG"]
        P5["5. START（最低）"]
    end
```

### 特异性优先级

当多条 PageRule 冲突时（例如一条 allow 一条 deny 同时匹配），按类型的优先级裁决：

```javascript
// server/core/auth.js:368
static _applyPageRuleSpecificity() {
    // 优先级：END > REGEX > EXACT > TAG > START
    // 相同类型按路径长度降序（更具体的路径优先）
    // 相同优先级时 deny 优先于 allow
}
```

### 默认配置

新安装时，"guest" 组的默认 PageRule：

```
match: START, path: "", roles: [read:pages, read:assets, read:comments, write:comments], deny: false
```

即默认所有页面对已登录用户可读。

## 5. 页面发布控制

除了权限模型，Wiki.js 还提供**发布调度**机制，控制页面内容的可见性：

```javascript
// server/controllers/common.js:467
let pageIsPublished = page.isPublished
if (pageIsPublished && page.publishStartDate) {
    pageIsPublished = moment(page.publishStartDate).isBefore(moment.utc())
}
if (pageIsPublished && page.publishEndDate) {
    pageIsPublished = moment(page.publishEndDate).isAfter(moment.utc())
}
if (!pageIsPublished && !hasWritePermission) {
    // 403 Unauthorized
}
```

三个字段控制发布状态：

| 字段 | 类型 | 说明 |
|------|------|------|
| `isPublished` | boolean | 是否已发布 |
| `publishStartDate` | datetime | 定时上架时间 |
| `publishEndDate` | datetime | 定时下架时间 |

## 6. 导航可见性

导航树（侧边栏）支持独立的可见性控制：

```graphql
# server/graph/schemas/navigation.graphql
type NavigationItem {
    # ...
    visibilityMode: NavigationVisibilityMode  # all | groups
    visibilityGroups: [Int]                    # 可见的用户组 ID 列表
}

enum NavigationVisibilityMode {
    all     # 所有人可见
    groups  # 仅指定组可见
}
```

除了导航级别的控制，页面树（`pageTree`）中的每个节点也会通过 `checkAccess()` 进行 `read:pages` 权限检查，确保用户在树上只能看到自己有权限查看的页面。

## 7. JWT 令牌设计

### 令牌结构

Wiki.js 使用 RS256 签名的 JWT 来维持用户会话：

```javascript
// server/models/users.js:225
static async refreshToken(user) {
    const userGroups = await user.getGroups()

    const token = jwt.sign(
        {
            id: user.id,
            email: user.email,
            name: user.name,
            permissions: userGroups.flatMap(g => g.permissions),
            groups: userGroups.map(g => g.id)
        },
        privateKey,
        {
            algorithm: 'RS256',
            issuer: 'wiki.js',
            expiresIn: WIKI.config.auth.tokenExpiration
        }
    )
}
```

### 令牌吊销机制

JWT 令牌带有 `u`（用户级）和 `g`（组级）版本号。当用户被移除出某个组，或组的权限发生变化时，对应的版本号递增，已签发的令牌在下次请求时被判定为无效，需要刷新。

### 完整权限查询

```mermaid
sequenceDiagram
    participant User as 用户请求
    participant JWT as JWT Middleware
    participant Auth as 权限引擎
    participant Page as 页面查询

    User->>JWT: 请求 + JWT Cookie
    JWT->>JWT: 验证签名/过期/吊销版本
    JWT-->>User: 用户身份 + permissions 数组

    User->>Auth: checkAccess(user, [read:pages], pageObj)
    Auth->>Auth: 检查 manage:system 绕过
    Auth->>Auth: 检查 read:pages 在 permissions 中
    Auth->>Auth: 遍历 PageRule 匹配 pageObj.path/locale/tags
    Auth-->>User: ✅/❌

    User->>Page: 查询页面数据
    Page->>Auth: getEffectivePermissions(req, page)
    Auth-->>Page: 页面级权限映射（可读/可写/可评论/可看源码等）
    Page-->>User: 按权限裁剪后的页面数据
```

---

> **GitHub:** [requarks/wiki](https://github.com/requarks/wiki)
