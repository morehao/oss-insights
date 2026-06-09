# Outline 文档多版本机制深度分析

## 概述

Outline 的版本管理围绕 **Revision（修订版）** 概念构建，采用**全量快照（Full Snapshot）**模式——每次值得记录的变更都创建一份完整的 Prosemirror 文档状态快照。与增量差异（diff）存储相比，全量快照的优点是读取速度快（无需合并历史）、历史浏览简单，代价是存储空间线性增长。

> 本文基于 v1.8.1 源码分析，核心代码位于 `server/`（后端）和 `shared/editor/`（差异对比引擎）。

## 版本存储模型

### Revision 数据模型

`server/models/Revision.ts` 定义了 Revision 的结构：

| 字段 | 类型 | 说明 |
|------|------|------|
| `title` | `string` | 修订时的文档标题 |
| `content` | `JSONB` | 完整 Prosemirror 文档状态（节点树 JSON） |
| `text` | `TEXT` | 已废弃的历史字段 |
| `icon` / `color` | `string` | 修订时的图标和颜色 |
| `name` | `string\|null` | 用户可选的修订名称 |
| `userId` | `UUID` | 触发此修订的用户 |
| `collaboratorIds` | `UUID[]` | 此修订窗口内的所有协作者 |
| `editorVersion` | `string` | 编辑器版本标识 |
| `version` | `SMALLINT` | 文档版本号 |
| `documentId` | `UUID` | 关联文档 |

**每次创建的完整快照**（`revisionCreator.ts`）：

```typescript
static buildFromDocument(document: Document) {
  return this.build({
    title: document.title,
    content: document.content,     // 完整 Prosemirror JSON
    icon: document.icon,
    color: document.color,
    editorVersion: document.editorVersion,
    version: document.version,
    collaboratorIds: [],  // 从 Redis 填充
  });
}
```

### 文档状态与数据流

```
ProseMirror 编辑器状态 (JSON 节点树)
  ↓ 序列化
Document.content (JSONB 数据库列)
  ↓ 事件触发
Revision.content (JSONB) ← 完整拷贝，不是 diff
  ↓
ChangesetHelper.getChangeset(revision1.content, revision2.content)
  ↓
Diff 插件渲染高亮
```

## 版本创建时序

### 触发事件

版本创建通过三种事件触发，由 `server/queues/processors/RevisionsProcessor.ts` 处理：

```
事件类型                   触发条件               Revision 创建
──────────────────────────────────────────────────────────────
documents.publish          文档首次发布/重新发布     ✅ 创建
documents.update.debounced 编辑后 5 分钟去抖完成     ✅ 创建
documents.update           用户导航离开/关闭标签页   ✅ 仅 done: true 时创建
```

### 去抖机制（防止频繁创建 Revision）

```
编辑操作
  │
  ├── 前端 auto-save (3s 去抖)
  │    └── POST /api/documents.update → 写入数据库
  │
  ├── Bull 队列 documents.update 事件
  │    └── DebounceProcessor（5 分钟延迟）
  │         ├── 如果 5 分钟内又有新更新 → 取消上次延迟，重新排队
  │         └── 5 分钟后无新更新 → 发射 documents.update.debounced
  │
  └── RevisionsProcessor 收到 debounced 事件
       └── 对比最新 Revision → 如果内容无变化则跳过（去重）
```

**关键代码**（`server/queues/processors/DebounceProcessor.ts`）：

```typescript
// 将频繁的 documents.update 收敛为单个 documents.update.debounced
// 生产环境 5 分钟延迟，开发环境 30 秒
```

### 重复检测

即使触发了 Revision 创建，如果内容与上一个 Revision 相同则跳过（`server/queues/processors/RevisionsProcessor.ts:38-44`）：

```typescript
const previous = await Revision.findOne({ where: { documentId }, order: [["createdAt", "DESC"]] });
if (previous && isEqual(document.content, previous.content) && document.title === previous.title) {
  return;  // 内容无变化，不创建新 Revision
}
```

这意味着：快速保存但内容不变不会产生冗余快照。

## 文档生命周期

### 状态流转

```
编辑中 (Draft)
  │
  ├── publish() → 已发布 (Published)
  │
  ├── archive() → 已归档 (Archived, archivedAt 非空)
  │    └── restore() → 已发布
  │
  ├── destroy() → 已删除 (Deleted, deletedAt 非空)
  │    ├── restore() → 恢复删除前状态
  │    └── force destroy → 永久删除（不可恢复）
  │
  └── 归档后删除 → 已删除
```

### 归档

`Document.archiveWithCtx()`（`server/models/Document.ts:1170-1189`）：
- 从集合导航结构中移除文档
- 递归归档所有子文档（`archiveWithChildren` 设置 `archivedAt`）

### 删除

`Document.destroyWithCtx()`（`server/models/Document.ts:1244-1273`）：
- 先移除集合导航结构
- 调用 Sequelize 的 `destroy()`（Paranoid 模式，设置 `deletedAt`）
- 触发 `delete` 事件

### 永久删除

`server/commands/documentPermanentDeleter.ts`：
- 仅对已删除文档生效（`deletedAt` 非空）
- 搜索文档内容中的附件引用，对不再被引用的附件发起清理任务
- 清除子文档的 `parentDocumentId` 引用
- 每批处理 10 个文档，批次间 sleep 以避免锁竞争

### 恢复

`server/commands/documentRestorer.ts` 处理三类恢复：

1. **从回收站恢复** — 恢复 `deletedAt` 为 null，重新加入集合导航结构
2. **从归档恢复** — 递归恢复所有子文档的 `archivedAt` 为 null
3. **恢复到某 Revision** — 将 Revision 的 content/title/icon/color 复制回当前文档，创建一条新的 Revision

## 版本对比与 Diff 引擎

### Changeset 计算

`shared/editor/lib/ChangesetHelper.ts` 实现差异计算：

```typescript
ChangesetHelper.getChangeset(revisionContent, previousRevisionContent)
```

工作流程：

1. 基于编辑器扩展创建 Prosemirror Schema
2. 分别将两个 Revision 的 JSON 解析为 Prosemirror 文档节点
3. 使用 `prosemirror-recreate-transform` 逆向推导两次文档状态之间的变换
4. 将变换步骤输入 `prosemirror-changeset` 的 `ChangeSet.create()`
5. 合并交错的变更（如同一个单词的多次替换）为更清晰的块
6. 区分**内容变更**（插入/删除）和**属性变更**（如代码块语言切换、表格单元格类型）

### Diff 渲染

`shared/editor/extensions/Diff.ts` 是一个 Prosemirror 插件：

- 创建 `DecorationSet`，使用 CSS 类标记差异：
  - **绿色高亮** — 新增内容
  - **红色 + 删除线** — 删除内容（保留在原位作为 widget 展示）
  - **黄色高亮** — 属性修改（如代码块语言）
- 支持快捷键导航到下/上一个变更点（滚动到视口内）
- 处理块级节点（图片、表格等）的装饰
- 对表格单元格和列表做智能展开，避免冗余嵌套

### 虚拟 Latest Revision

`shared/utils/RevisionHelper.ts` 提供了一个特殊标识 `latest-{documentId}`，表示"当前文档的最新状态"。这使得用户可以：

- 对比任意 Revision 与当前文档状态
- 在历史列表中看到"当前版本"占位
- 在 UI 中标记当前版本与历史版本的区别

## 前端版本历史体验

### 历史侧边栏

`app/scenes/Document/components/History/History.tsx`：

历史列表将 Revision 与文档事件（发布、归档、删除、移动等）交织展示，按 `createdAt` 排序合并。如果当前文档状态与最新 Revision 不同，会插入一个虚拟的"当前版本"条目。

**时间线示例：**

```
[10:30] 当前版本（未保存的变更） ← 虚拟 Latest Revision
[10:25] 张三 编辑 · v3
[10:20] 系统 发布文档
[10:15] 李四 编辑 · v2
[10:10] 张三 创建文档 · v1
```

### Revision 查看器

`app/scenes/Document/components/RevisionViewer.tsx`：

- 从 URL 读取 `changes`（是否显示 diff）和 `compareTo`（对比目标）参数
- 创建包含 `Diff` 扩展的编辑器实例，只读展示
- 可选择"对比上一版本"或"对比指定版本"

### 变更高亮控制

`app/scenes/Document/components/History/HighlightChangesControl.tsx` 提供：

- 开关切换 Diff 高亮
- 版本选择下拉框（列出所有 Revision，按年份分组）

## API 接口

| 端点 | 功能 | 鉴权要求 |
|------|------|---------|
| `revisions.info` | 获取单个 Revision，或构建 Latest 虚拟 Revision | 文档读取权限 |
| `revisions.list` | 分页列出文档的 Revision 列表 | 文档读取权限 |
| `revisions.update` | 修改 Revision 名称 | 文档编辑权 + 本人或管理员 |
| `revisions.delete` | 软删除 Revision | 管理员 |
| `revisions.export` | 导出为 Markdown / HTML / PDF | 文档读取权限 |

## 设计权衡与局限

### 优势

1. **全量快照，读取快** — 访问任意历史版本无需合并计算，直接读取 `content` 列
2. **去抖 + 去重** — 5 分钟去抖窗口 + 内容等值检测，减少冗余快照；快速编辑不会产生过多垃圾
3. **虚拟 Latest 技术** — 用合成 ID 实现"当前状态"的参与比较，不需要额外存储
4. **事件与 Revision 混合展示** — 用户看到的不只是版本号，而是完整的文档事件时间线

### 已知局限

1. **存储无压缩** — Revision 是全量快照而非增量 diff，存储随编辑次数线性增长。如果采用增量 diff + 定期全量快照的混合策略，存储效率会更高
2. **5 分钟去抖窗口偏长** — 编辑 10 分钟后关闭标签页，只有最后 3-8 分钟的内容被保存为 Revision。中间的内容只有数据库中的当前版本
3. **协作者跟踪依赖 Redis** — `doc-collaborators:{documentId}` 键在 Revision 创建时被消费，如果 Redis 数据丢失，协作者归属信息会丢失
4. **永久删除的竞态条件** — 在检查 `deletedAt` 和真正 `force: true` destroy 之间存在窗口，文档可能在此期间被恢复

---

> **参考文件：**
> - `server/models/Revision.ts` — Revision 数据模型
> - `server/commands/revisionCreator.ts` — Revision 创建命令
> - `server/queues/processors/RevisionsProcessor.ts` — Revision 事件处理器
> - `server/queues/processors/DebounceProcessor.ts` — 去抖处理器
> - `server/commands/documentRestorer.ts` — 文档恢复命令
> - `server/commands/documentPermanentDeleter.ts` — 永久删除命令
> - `server/routes/api/revisions/revisions.ts` — Revision API 路由
> - `shared/editor/lib/ChangesetHelper.ts` — 差异计算引擎
> - `shared/editor/extensions/Diff.ts` — Diff 渲染插件
> - `shared/utils/RevisionHelper.ts` — Latest 虚拟版本工具
> - `app/scenes/Document/components/History/History.tsx` — 历史侧边栏
> - `app/scenes/Document/components/History/RevisionListItem.tsx` — Revision 列表项
> - `app/scenes/Document/components/RevisionViewer.tsx` — Revision 查看器
> - `app/scenes/Document/hooks/useDocumentSave.ts` — 前端自动保存逻辑
