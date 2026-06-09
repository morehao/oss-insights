# Outline 文档资源权限管理深度分析

## 概述

Outline 的权限体系以**层级继承**和**声明式策略**为两大支柱。与常见 RBAC（基于角色）模型不同，Outline 采用类似 Ruby CanCanCan 的能力授权模式——不关注用户"是什么角色"，而是关注用户"能做什么操作"。

> 本文基于 v1.8.1 源码分析，核心策略文件位于 `server/policies/`。

## 权限模型全景

### 层级结构

```
Team（团队）
 ├── 全局设置：sharing、memberCollectionCreate、memberTeamCreate
 │
 ├── Collection（集合，公开/私有）
 │    ├── permission: null (私有) | Read | ReadWrite | Admin
 │    ├── UserMembership[]（直接用户资格）
 │    ├── GroupMembership[]（组资格）
 │    │
 │    └── Document（文档）
 │         ├── UserMembership[]（覆盖集合级权限）
 │         ├── GroupMembership[]（覆盖集合级权限）
 │         ├── 分享链接（Share 模型）
 │         └── Attachment（附件，团队级 ACL）
```

### 关键类型定义

Outline 定义了 4 种用户角色（`shared/types.ts:2-7`）和两种资源权限级别：

```typescript
// 用户角色
enum UserRole { Admin = "admin", Member = "member", Viewer = "viewer", Guest = "guest" }

// 集合权限
enum CollectionPermission { Read = "read", ReadWrite = "read_write", Admin = "admin" }

// 文档权限（与集合相同）
enum DocumentPermission { Read = "read", ReadWrite = "read_write", Admin = "admin" }
```

### 权限检查流程（以读取文档为例）

```
1. 路由层: authorize(actor, "read", document)
       ↓
2. cancan.can(): 遍历已注册的能力规则
       ↓
3. 匹配到 document.ts: allow(User, "read", Document, condition)
       ↓
4. 执行 condition 函数，逐层检查：
   ┌─ isTeamModel(actor, document) → 必须同团队
   ├─ includesMembership(document, [Read, ReadWrite, Admin]) → 直接/组成员资格？
   ├─ document.isDraft && actor.id === document.createdById → 自己的草稿？
   └─ can(actor, "readDocument", document?.collection) → 集合级权限？
       └─ collection.ts: allow(User, "readDocument", Collection)
          ├─ user.isAdmin → true（管理员绕过）
          ├─ collection.isPrivate / user.isGuest → 需要成员资格
          └─ 公开集合 → true
```

## Cancan 授权引擎

### 核心设计

`server/policies/cancan.ts` 定义了 `CanCan` 类，提供三个核心方法：

```typescript
// 注册能力规则 — 谁可以对什么目标做什么操作，在什么条件下
allow(User, "read", Document, (actor, document) => {
  // 返回 true/false
});

// 检查能力 — 用户可以操作这个目标吗？
can(actor, "read", document);  // → boolean

// 授权检查 — 不能操作则抛 AuthorizationError
authorize(actor, "read", document);  // → void | throws
```

### 规则注册机制

所有 `allow()` 调用在**模块导入时执行**。`server/policies/index.ts` 导入所有策略文件，应用启动时一次性注册所有能力规则：

```typescript
// server/policies/index.ts（简化）
import "./document";
import "./collection";
import "./share";
import "./attachment";
// ... 每个策略文件导入即执行 allow() 注册
```

### 条件函数的组合模式

策略文件使用组合函数构建条件逻辑（`server/policies/utils.ts`）：

```typescript
// and() — 所有条件为真
function and(...args) { return args.every(Boolean) && args; }

// or() — 首个真值
function or(...args) { return args.find(Boolean); }

// isTeamModel() — 同团队检查
function isTeamModel(actor, model) { return actor.teamId === model.teamId; }
```

### 策略示例：集合读取

```typescript
// server/policies/collection.ts:34-47
allow(User, "read", Collection, (user, collection) => {
  if (!collection || user.teamId !== collection.teamId) return false;
  if (user.isAdmin) return true;                  // 管理员无条件读取
  if (collection.isPrivate || user.isGuest) {     // 私有集合需要资格
    return includesMembership(collection, Object.values(CollectionPermission));
  }
  return true;  // 公开集合，团队成员可读
});
```

关键设计决策：**管理员总是可以读取任何集合**（`if (user.isAdmin) return true`），即使集合是私有的。这是设计上对管理员的信任假设。

### 策略示例：文档读取

```typescript
// server/policies/document.ts:17-30
allow(User, "read", Document, (actor, document) =>
  and(
    isTeamModel(actor, document),
    or(
      includesMembership(document, [Read, ReadWrite, Admin]),  // 直接资格
      and(!!document?.isDraft, actor.id === document?.createdById),  // 自己的草稿
      can(actor, "readDocument", document?.collection)  // 集合级权限遍历
    )
  )
);
```

注意第三分支 `can(actor, "readDocument", document?.collection)`——文档权限检查会**递归向上**检查集合权限，这就是层级继承的实现方式。

## 成员资格与继承机制

### 用户成员资格

`UserMembership` 模型（`server/models/UserMembership.ts`）管理单个用户在集合或文档上的权限：

- **`collectionId`** — 作用于集合（无 `documentId` 时）
- **`documentId`** — 作用于文档（无 `collectionId` 时）
- **`sourceId`** — 如果非空，表示这是一个"派生"资格，来自上级资源的级联
- **`permission`** — `Read` / `ReadWrite` / `Admin`

### 组成员资格

`GroupMembership` 模型结构与 `UserMembership` 完全相同，但作用于组而非单个用户。组概念允许对用户分组批量授权，减少管理成本。

### 级联机制（核心设计）

当在集合或文档上创建/更新成员资格时，Sequelize 的 `AfterCreate` 和 `AfterUpdate` 钩子自动将其传播到所有子文档：

```typescript
// server/models/UserMembership.ts:206-214
@AfterCreate
static async createSourcedMemberships(model, options) {
  if (model.sourceId || !model.documentId) return;  // 仅处理根资格
  return this.recreateSourcedMemberships(model, options);
}

@AfterUpdate  // server/models/UserMembership.ts:237-261
static async updateSourcedMemberships(model, options) {
  if (model.sourceId || !model.documentId) return;
  if (model.changed("permission")) {
    // 批量更新所有派生资格
    await this.update({ permission: model.permission }, {
      where: { sourceId: model.id },
      transaction,
    });
  }
}
```

**级联流程：**

```
在集合 A 上设置 UserMembership(ReadWrite)
  ↓
UserMembership.createSourcedMemberships()
  ↓
找到集合 A 下所有文档（递归子文档）
  ↓
为每个文档创建 UserMembership(ReadWrite, sourceId=rootId)
  ↓
当根资格 permission 变更 → 批量更新所有 sourceId 指向它的派生资格
  ↓
当根资格被删除 → 批量删除所有派生资格
```

这确保了**权限变更在文档树中自上而下传递**，无需手动逐文档调整。

## 分享链接系统

### Share 模型

`server/models/Share.ts` 管理文档和集合的公开分享链接。

| 字段 | 说明 |
|------|------|
| `published` | 分享链接是否启用 |
| `revokedAt` | 撤销时间（非空 = 已撤销） |
| `includeChildDocuments` | 是否包含子文档 |
| `urlId` | 短链接标识 |
| `domain` | 自定义域名 |
| `allowIndexing` | 是否允许搜索引擎索引 |
| `allowSubscriptions` | 是否允许邮件订阅变更 |

### 分享策略

```typescript
// server/policies/share.ts:30-40
allow(User, "update", Share, (actor, share) =>
  and(
    !actor.isGuest,
    !actor.isViewer,
    or(
      isTeamAdmin(actor, share),
      can(actor, "share", share?.collection) ||  // 集合级分享权限
      can(actor, "share", share?.document)        // 文档级分享权限
    )
  )
);
```

分享的实际检查委托给了对应的集合/文档策略中的 `share` 能力——确保"能分享"的前提是"能读取"且分享功能已开启。

### 公开访问流程

`server/commands/shareLoader.ts:23-138` 中的 `loadPublicShare` 负责处理公开链接的访问：

1. 通过 `id` 或 `urlId` + `teamId` 查找分享记录
2. 验证分享未撤销、已发布
3. 验证团队和集合的 `sharing` 开关均开启
4. 验证文档未归档
5. 如果 `includeChildDocuments` 为 true，构建导航树
6. 试用版团队被拦截

## 附件权限

### 设计决策：团队级而非文档级

附件权限是 Outline 权限模型中最值得讨论的设计选择。

`server/policies/attachment.ts` 的权限规则很简单：

```typescript
allow(User, "createAttachment", Team, isTeamModel);            // 团队成员可上传
allow(User, ["read", "update", "delete"], Attachment, (actor, attachment) =>
  and(isTeamModel(actor, attachment), or(actor.isAdmin, isOwner(actor, attachment)))
);  // 管理员或上传者可读/改/删
```

在附件重定向接口中（`server/routes/api/attachments/attachments.ts:279-286`），实际的访问控制更加宽松：

```typescript
// Private attachments are accessible to any member of the workspace they
// belong to. This is intentional and not a permission bypass – attachments
// are owned by the workspace (team), not by individual documents. Checking
// document-level permissions here would be insufficient anyway as attachments
// can exist independently of documents.
if (attachment.isPrivate && attachment.teamId !== user?.teamId) {
  throw AuthorizationError();
}
```

这意味着：**同团队的任何成员都可以访问任意附件**，无论该附件关联的文档是否对该用户有权限。项目注释明确说明了这是有意为之——附件是"团队资产"而非"文档资产"。

### ACL 类型

附件模型中的 `acl` 字段控制存储层的访问策略：

- **`public-read`** — 附件存储在 S3 的公开桶中，通过 `canonicalUrl` 直接访问
- **`private`** — 附件存储在私有桶中，通过 `redirectUrl` 获取预签名 URL（有时效性）

## 组合模式与工具函数

策略文件中的条件逻辑大量使用组合函数，这是一种**函数式策略组合**模式：

```typescript
// server/policies/utils.ts
function and(...args) {
  return args.every(Boolean) ? args : false;
}

function or(...args) {
  return args.find(Boolean);
}

function isTeamModel(actor, model) {
  return actor.teamId === model.teamId;
}

function isOwner(actor, model) {
  return actor.id === model.userId || actor.id === model.createdById;
}

function isTeamAdmin(actor, model) {
  return isTeamModel(actor, model) && actor.isAdmin;
}
```

这种组合方式让策略条件的可读性极高——一条策略声明就是一行自然语言风格的描述：

```typescript
allow(User, "delete", Document, (actor, document) =>
  and(
    !document.isDeleted, isTeamModel(actor, document), isTeamMutable(actor),
    or(can(actor, "unarchive", document), can(actor, "update", document))
  )
);
```

## 设计权衡与局限

### 优势

1. **声明式策略** — 每条能力规则清晰独立，新增资源只需新增一个策略文件
2. **级联继承** — 权限变更自动传播到文档树，减少管理负担
3. **向上递归** — 文档权限检查可递归到集合，实现了层级继承而无需复制数据
4. **组合模式** — `and/or/isTeamModel/isOwner` 组合器使条件可读性极高

### 已知局限

1. **附件权限是团队级** — 任何团队成员可通过 URL 访问任意附件，文档级权限无法保护附件
2. **成员资格无过期机制** — `UserMembership` 和 `GroupMembership` 没有过期时间，访问权限持续到被手动撤销
3. **`isTeamMutable` 是桩代码** — `server/policies/utils.ts:100-102` 始终返回 `true`，只读部署的支持不完整
4. **级联加载要求** — `includesMembership()` 要求预加载 `memberships` 和 `groupMemberships`，遗漏会触发 `invariant` 运行时错误
5. **集合私有性判定** — `collection.isPrivate` 通过 `!this.permission` 实现（`permission === null` 为私有），语义隐晦

---

> **参考文件：**
> - `server/policies/cancan.ts` — CanCan 授权引擎核心
> - `server/policies/document.ts` — 文档策略（303 行，22 条规则）
> - `server/policies/collection.ts` — 集合策略（241 行，15 条规则）
> - `server/policies/share.ts` — 分享策略
> - `server/policies/attachment.ts` — 附件策略
> - `server/policies/utils.ts` — 条件组合器
> - `server/models/UserMembership.ts` — 用户成员资格与级联钩子
> - `server/models/GroupMembership.ts` — 组成员资格与级联钩子
> - `server/models/Share.ts` — 分享模型
> - `server/commands/shareLoader.ts` — 公开分享加载
> - `server/routes/api/attachments/attachments.ts:279-286` — 附件访问控制
