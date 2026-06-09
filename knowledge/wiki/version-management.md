# Wiki.js 多版本文档机制详解

> **关联文档:** [Wiki.js 主解读](./README.md) · [文档资源权限管理](./permission-model.md)

## 1. 双轨版本存储架构

Wiki.js 的版本管理采用**双轨架构**，两条轨道各有侧重，协同工作：

| 维度 | PageHistory 表 | Git 存储 |
|------|---------------|----------|
| **定位** | 操作级快照，用于 Web 前端历史查看和版本恢复 | 外部持久化，用于备份、差异查看和外部协作 |
| **存储位置** | 主数据库 `pageHistory` 表 | 外部 Git 仓库（GitHub/GitLab 或自建） |
| **粒度** | 每次变更一条记录，含完整页面内容 | 每次变更一个 commit，含文件变更 |
| **触发时机** | update / move / delete / restore | 配置为 storage 后端后，每次页面保存 |
| **保留策略** | 按时间清理（`purge()`），可配置 | Git 仓库全量保留 |
| **访问方式** | GraphQL 查询 `history(id)` / `version(pageId, versionId)` | Git 原生工具（git log/diff/clone） |

```mermaid
graph TB
    subgraph "页面操作"
        Update[更新页面]
        Move[移动页面]
        Delete[删除页面]
        Restore[恢复版本]
    end

    subgraph "PageHistory 轨道"
        AddVersion[addVersion 创建快照]
        HistoryDB[(pageHistory 表)]
        GetHistory[getHistory 查询历史]
        Purge[purge 清理旧版本]
    end

    subgraph "Git 存储轨道"
        GitStorage[storage/git 模块]
        GitRepo[(外部 Git 仓库)]
        GitDiff[git diff/log]
    end

    Update --> AddVersion
    Move --> AddVersion
    Delete --> AddVersion
    Restore --> AddVersion

    AddVersion --> HistoryDB
    HistoryDB --> GetHistory
    HistoryDB --> Purge

    Update --> GitStorage
    Move --> GitStorage
    Delete --> GitStorage
    GitStorage --> GitRepo
    GitRepo --> GitDiff
```

## 2. PageHistory 快照机制

### 触发时机

`WIKI.models.pageHistory.addVersion()` 在以下操作中被调用：

- **`pages.updatePage()`** — 页面内容/标题/路径/标签变更前
- **`pages.movePage()`** — 页面移动至新路径前
- **`pages.deletePage()`** — 页面删除前
- **`PageMutation.restore()`** — 版本恢复前

```javascript
// server/models/pageHistory.js:50
static async addVersion(opts) {
    // 获取当前页面最新状态
    // 复制关键字段：path, hash, title, description,
    //   isPublished, publishStartDate, publishEndDate,
    //   content, contentType
    // 记录操作者和时间
    // 写入 pageHistory 表
}
```

### Action 类型检测

`getHistory()` 通过比较相邻版本的信息来推导操作类型：

```mermaid
graph LR
    V1[(版本 N-1)] --> Compare{路径是否变化？}
    Compare -->|是| ActionMove[move]
    Compare -->|否| ActionEdit[edit]
```

- **`initial`** — 第一条历史记录（页面创建时的自动快照）
- **`edit`** — 路径未变，仅内容/标题等变更
- **`move`** — 路径发生变化

### 分页查询

```javascript
// server/models/pageHistory.js:142
static async getHistory({ pageId, offsetPage, offsetSize }) {
    // 按 pageId 查询该页面的所有版本
    // 按 createdAt 降序排列
    // 分页返回（offsetPage/offsetSize）
    // 每个版本附带 author 信息
    // 遍历推导 action 类型
}
```

### 快照存储字段

每次快照完整保存以下字段，确保恢复时能还原页面完整状态：

`pageHistory` 表字段：`id`, `pageId`, `path`, `hash`, `title`, `description`, `isPublished`, `publishStartDate`, `publishEndDate`, `content`, `contentType`, `authorId`, `editorKey`, `localeCode`, `action`, `versionDate`, `createdAt`

## 3. Git 存储集成

当配置 `storage/git` 作为存储后端时，每次页面变更会同步到 Git 仓库：

```mermaid
sequenceDiagram
    participant Page as 页面操作
    participant Storage as storage/git
    participant Git as Git 仓库

    Page->>Storage: 内容变更推送
    Storage->>Git: git add <文件>
    Storage->>Git: git commit -m "page commit"
    Storage->>Git: git push
    Git-->>Storage: ✅ 推送成功

    Note over Storage,Git: commit message 格式：<br/>[${pageId}] ${title}<br/><br/>locale: ${localeCode}<br/>path: ${path}<br/>action: ${action}
```

Git 存储的典型目录结构：

```
wiki-repo/
├── <locale>/
│   ├── <page-path>.md
│   └── ...
└── ...
```

## 4. 版本恢复流程

```mermaid
sequenceDiagram
    participant Admin as 管理员
    participant GQL as GraphQL
    participant Hist as PageHistory
    participant Page as 页面模型
    participant Stg as 存储

    Admin->>GQL: mutation restore(pageId, versionId)
    GQL->>Hist: getVersion(pageId, versionId)
    Hist-->>GQL: 历史版本数据
    GQL->>Hist: addVersion() 先保存当前版本快照
    GQL->>Page: updatePage(...)
    Page->>Stg: 同步到存储后端
    Page-->>GQL: 页面已恢复
    GQL-->>Admin: 恢复成功
```

## 5. 历史清理策略

```javascript
// server/models/pageHistory.js:225
static async purge(olderThan) {
    // olderThan: ISO 8601 时长字符串，如 "P30D" (30天)
    // 删除 createdAt < (now - olderThan) 的历史记录
    await WIKI.models.pageHistory
        .query()
        .where('createdAt', '<', moment.utc().subtract(iso8601ToDuration(olderThan)).toDate())
        .del();
}
```

清理通过后台定时任务（`server/jobs/`）周期性执行，可在管理界面配置保留时长。

---

> **GitHub:** [requarks/wiki](https://github.com/requarks/wiki)
