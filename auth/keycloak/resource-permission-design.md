# 资源权限设计 — 从 Keycloak 提炼的通用方案

> 本文分析 Keycloak 的细粒度授权系统（Fine-Grained Authorization），提炼为语言无关的通用资源权限设计方案。

---

## 零、贯穿全文的示例场景

为便于理解，后续所有章节使用同一个业务场景举例：

> **企业文档管理系统**，有以下资源、角色和权限需求：

**资源清单：**

| 资源 | 类型 | 所有者 | 说明 |
|------|------|--------|------|
| `proj-spec-v2` | `project-doc` | 部门 A | 项目规格书 |
| `api-design` | `tech-doc` | 部门 A | API 设计文档 |
| `employee-reviews` | `hr-doc` | HR 部门 | 员工绩效评估 |

**作用域（Scope）：** `read`（阅读）、`write`（编辑）、`approve`（审批发布）、`delete`（删除）

**用户与角色：**

| 用户 | 角色 | 所属 |
|------|------|------|
| Alice | `editor` | 部门 A |
| Bob | `senior-engineer` | 部门 A |
| Charlie | `hr-manager` | HR 部门 |
| Dave | `admin` | 系统管理员 |

**权限需求：**
1. 部门 A 的所有成员对 `project-doc` 和 `tech-doc` 有 `read` 权限
2. `senior-engineer` 才能 `approve` project-doc
3. `hr-doc` 仅 HR 部门成员可读写
4. `admin` 拥有所有资源的所有权限
5. 文档作者可以自主授权他人访问自己的文档（UMA）

---

## 一、核心模型

资源权限系统围绕四个核心概念构建：

```
资源 (Resource)  ──── 作用域 (Scope)
   │                      │
   └──────┬───────────────┘
          ▼
        策略 (Policy) ──── 子策略 (Policy 递归)
          │
          ▼
       权限 (Permission) — 资源 + 作用域 + 策略的评估结果
```

### 1.1 资源（Resource）

受保护的业务对象。关键属性：

| 属性 | 说明 | 必需 |
|------|------|------|
| `id` | 唯一标识 | 是 |
| `name` | 资源名称（同一资源服务器下唯一） | 是 |
| `type` | 资源类型，用于按类型批量授权 | 否 |
| `uris` | 一组 URI 标识 | 否 |
| `owner` | 资源所有者 ID | 是 |
| `attributes` | 扩展属性键值对 | 否 |

资源通过 type 实现分层控制：可以按类型（如 `project-doc`, `hr-doc`）定义通用策略，也可以按单个实例（如 `proj-spec-v2`）定义专属策略。

> **场景示例：** 三份文档分别属于 `project-doc`、`tech-doc`、`hr-doc` 三种资源类型。管理员只需为每个类型配置一套策略，而非为每份文档逐一配置。

### 1.2 作用域（Scope）

资源上可执行的操作。典型的 scopes：

| 粒度 | 示例 |
|------|------|
| CRUD | `create`, `read`, `update`, `delete` |
| 业务动作 | `approve`, `publish`, `archive`, `transfer` |
| 访问级别 | `view`, `edit`, `admin` |

Scope 可以跨资源复用。一个资源可以关联多个 scopes，一个 scope 也可以服务于多个资源。

> **场景示例：** `project-doc` 关联 `read` + `write` + `approve` + `delete` 四个 scopes。`hr-doc` 只关联 `read` + `write`（不允许非 HR 人员审批或删除人事档案）。`approve` scope 在不同类型资源上的权限门槛不同。

### 1.3 策略（Policy）

授权规则。这是整个系统的核心——策略决定谁能访问什么。策略支持组合嵌套（父策略包含多个子策略，递归评估）。

每个策略包含：

| 属性 | 说明 | 场景示例 |
|------|------|----------|
| `name` | 策略名称 | `"仅 senior-engineer 可审批 project-doc"` |
| `type` | 策略类型（见 2.1） | `resource`（权限策略）、`role`（子策略） |
| `logic` | 逻辑模式：POSITIVE 或 NEGATIVE | NEGATIVE：`"不在黑名单中的允许"` |
| `config` | 策略特有配置 | 角色策略的 config 包含角色 ID 列表 |
| `associatedPolicies` | 子策略集合（组合模式，可递归） | 一个 resource 权限策略下嵌套角色策略和时间策略 |

> **场景示例：** 需求"仅 senior-engineer 可审批 project-doc"可以建模为：
> ```
> 权限策略 P1 (type=resource)
>   ├── 绑定资源类型: project-doc
>   ├── 绑定 scope: approve
>   └── 关联子策略:
>       └── 角色策略 R1 (type=role)
>           └── 检查: identity.hasRole("senior-engineer")?
> ```

### 1.4 资源服务器（Resource Server）

权限系统的**范围上下文**。通常映射为一个应用/微服务。资源、作用域、策略都在某个资源服务器下定义。

关键配置：

| 配置 | 可选值 | 说明 | 场景示例 |
|------|--------|------|----------|
| `policyEnforcementMode` | ENFORCING / PERMISSIVE / DISABLED | 无策略匹配时的默认行为 | 文档管理设为 **ENFORCING**：未配置策略的资源默认拒绝访问 |
| `decisionStrategy` | AFFIRMATIVE / UNANIMOUS / CONSENSUS | 多条策略冲突时的合并规则 | 文档审批设为 **UNANIMOUS**：需主管和法务都通过才算通过 |
| `allowRemoteManagement` | true / false | 是否允许通过 API 远程管理资源 | 开放平台设为 true，允许第三方通过 API 注册资源 |

---

## 二、策略系统

### 2.1 策略类型体系

策略分为三层：

```
权限策略 (Permission Policy) ── 桥接层：绑定资源+作用域到子策略
    │
    ├── resource: 绑定具体资源或资源类型
    ├── scope:    绑定作用域
    └── uma:      用户自管理（所有者授权）
    │
    ▼
子策略 ── 实际判定层：检查身份、上下文等条件
    │
    ├── role:        检查角色（支持角色继承）
    ├── user:        检查用户 ID
    ├── group:       检查组归属
    ├── client:      检查客户端 ID
    ├── client-scope: 检查客户端作用域归属
    ├── time:        检查时间范围
    ├── regex:       正则匹配自定义属性（如 claim 值匹配规则）
    ├── aggregate:   聚合策略（组合多个子策略，自带独立决策收集器）
    ├── js:          自定义脚本（最灵活）
    └── ...          可 SPI 扩展
```

关键理解：**权限策略**不直接做判定，它负责确定"哪些资源/作用域受保护"，然后委托给**子策略**做实际判定。子策略也可以进一步嵌套其他子策略。

> **场景示例：** 回看 Bob（senior-engineer）审批 project-doc 的场景：
> ```
> 权限策略 P1 (type=resource)
>   ├── resources: [类型=project-doc]
>   ├── scopes: [approve]
>   └── associatedPolicies:
>       └── 角色策略 (type=role)
>           └── config: { roles: ["senior-engineer"] }
> ```
> 评估时，P1 识别出"这是对 project-doc 的 approve 请求"，然后委托给角色策略判定"用户是否有 senior-engineer 角色"。角色策略只回答 yes/no，不关心什么资源。

### 2.2 策略评估流程

```
请求访问资源 R 的作用域 S
  │
  ├─ 1. 查找匹配策略
  │     ├── 按资源实例 R 查找 → 通过 resource_policy 表直接匹配 resource_id
  │     ├── 按资源类型查找 → 通过 policy_config 表的 defaultResourceType 匹配资源 type 字段
  │     └── 按作用域 S 查找 → 找到权限策略 P3
  │
  ├─ 2. 评估每个匹配策略
  │     └── 权限策略 P1
  │           ├── 遍历关联子策略
  │           │   ├── 角色策略 → identity.hasRole("admin")? → grant/deny
  │           │   ├── 时间策略 → now in [09:00, 17:00]? → grant/deny
  │           │   └── JS 策略 → 自定义逻辑
  │           └── 根据子策略的 DecisionStrategy 合并结果
  │
  └─ 3. 返回最终决策
```

> **场景示例：** Alice（editor）请求读取 `proj-spec-v2`（type=project-doc, scope=read）
> ```
> Step 1 — 查找匹配策略：
>   ├── 按资源实例: 无（proj-spec-v2 没有专属策略）
>   ├── 按资源类型: 找到 P1（project-doc 类型策略）
>   │     └── 关联子策略: 角色策略 R1（检查 editor 角色）
>   └── 按作用域: 找到 PS1（read scope 策略）
>         └── 关联子策略: 无（read 是基础权限，PS1 本身允许）
>
> Step 2 — 评估：
>   ├── P1 委托 R1 → Alice.hasRole("editor")? → 是 → grant
>   └── PS1 → 无条件允许 → grant
>
> Step 3 — 合并：AFFIRMATIVE → 允许 read
> ```

### 2.3 递归评估组件（AbstractPermissionProvider 模式）

```
function evaluatePermission(parentPolicy):
    for each associatedPolicy in parentPolicy.associatedPolicies:
        // 查缓存
        if cachedDecision exists:
            apply cached decision
            continue
        
        // 获取对应的策略评估器
        provider = getProvider(associatedPolicy.type)
        
        // 递归评估
        provider.evaluate(associatedPolicy)
        
        // 无明确决策时默认拒绝
        if effect == null:
            effect = DENY
        
        cache decision
```

这是**组合模式**的典型应用：权限策略树可以任意深度嵌套。

---

## 三、决策合并规则

### 3.1 决策策略（DecisionStrategy）

当策略树中有多个子策略时，合并规则有 3 种：

| 策略 | 规则 | 适用场景 | 场景示例 |
|------|------|----------|----------|
| **AFFIRMATIVE** | 任一子策略允许 → 最终允许 | 宽松控制，"管理员或所有者均可" | 文档删除：管理员或文档作者任一同意即可 |
| **UNANIMOUS** | 所有子策略都允许 → 最终允许 | 严格安全，"角色+时间+IP 全满足" | 财务文档审批：需同时满足 senior-engineer 角色 + 工作时间 |
| **CONSENSUS** | 允许数 > 拒绝数 → 最终允许；平局→拒绝 | 多评审人投票 | 技术评审：3 位架构师中至少 2 位同意 |

### 3.2 逻辑反转（Logic）

| 逻辑 | 效果 | 示例 |
|------|------|------|
| POSITIVE | 策略允许→最终允许，拒绝→最终拒绝 | 正常使用 |
| NEGATIVE | 策略允许→最终拒绝，拒绝→最终允许 | 黑名单场景（"不在黑名单中的允许"） |

### 3.3 强制模式（PolicyEnforcementMode）

| 模式 | 行为 | 场景 |
|------|------|------|
| ENFORCING | 无匹配策略时**拒绝** | 生产环境，默认安全 |
| PERMISSIVE | 无匹配策略时**放行** | 开发/测试，逐步迁移 |
| DISABLED | 跳过所有策略，**全部放行** | 性能压测、紧急开关 |

### 3.4 Scope 级别精细控制

权限评估结果不仅仅是"允许/拒绝"二值，而是**每个 scope 独立判定**：

```
请求资源 R 的 [read, write, delete]
  │
  ├── 策略 P1：允许 read（关联 scope=read）
  ├── 策略 P2：拒绝 delete（关联 scope=delete）
  └── 策略 P3：允许 write（关联 scope=write）
      │
      ▼
  最终结果：允许 [read, write]，拒绝 [delete]
```

这个设计让 scope 级别的授权粒度非常细。

> **场景示例：** Alice（editor）请求操作 `proj-spec-v2`：
> ```
> 请求: [read, write, approve, delete]
>
> ├── read:   类型策略 P_project 允许 editor → 允许
> ├── write:  类型策略 P_project 允许 editor → 允许
> ├── approve: 角色策略 R_senior 检查 → Alice 不是 senior-engineer → 拒绝
> └── delete:  仅管理员策略 → Alice 不是 admin → 拒绝
>
> 最终: 允许 [read, write]，拒绝 [approve, delete]
> ```

---

## 四、UMA（用户自管理权限）

标准 RBAC/ABAC 由管理员配置策略，但 UMA（User-Managed Access）让**资源所有者**自主授权：

```
资源所有者
  │
  ├─ 1. 创建资源，设置 ownerManagedAccess=true
  │
  ├─ 2. 用户 Eve 请求访问 → 生成 PermissionTicket
  │     { resource: "my-notes", scope: "read", requester: "Eve" }
  │
  ├─ 3. 资源所有者收到通知 → 审批票证
  │
  └─ 4. 票证标记为 granted → 用户 B 获得访问权限
```

`PermissionTicket` 模型：

| 字段 | 说明 |
|------|------|
| `resource` | 被请求的资源 |
| `scope` | 请求的作用域（可为 null = 全部） |
| `requester` | 请求者 ID |
| `owner` | 资源所有者 ID |
| `granted` | 是否已授权 |
| `policy` | 授权后关联的策略 |

UMA 的典型场景：文档共享系统、资源协作平台、社交应用的内容授权。

> **场景示例：** Alice 创建了个人项目笔记 `my-notes`，需要让实习生 Eve 查看：
> ```
> Alice 创建资源 my-notes → 设置 ownerManagedAccess=true
> Eve 请求读取 my-notes → 系统生成 PermissionTicket
> Alice 收到审批通知 → 批准票证 → 系统自动创建 UMA 权限策略
> Eve 再次请求 → 命中 UMA 策略 → 允许 read
> ```

---

## 五、细粒度管理权限（FGAP）

大型系统需要分级管理：超级管理员 → 资源管理员 → 普通用户。FGAP（Fine-Grained Admin Permissions）将 admin 权限也纳入这套模型管理：

```
FGAP 的核心思想：
  admin 操作（创建用户、修改角色、删除客户端）也视为"资源"
  admin 人员的角色/身份也视为"策略条件"
  同一套评估引擎同时服务：业务授权 + 管理授权
```

FGAP 的关键设计：

1. **管理操作映射为资源类型**：将 Keycloak 管理对象映射为 5 种资源类型，每种类型对应一组操作 scope：

| 资源类型 | 操作 scope |
|----------|-----------|
| `Clients` | `manage`, `view`, `map-roles`, `map-roles-client-scope`, `map-roles-composite` |
| `Groups` | `manage`, `view`, `manage-members`, `manage-membership` |
| `Roles` | `manage`, `view`, `map-roles` |
| `Users` | `manage`, `view`, `impersonate`, `map-roles`, `manage-group-membership`, `reset-password` |
| `Organizations` | `manage`, `view`, `manage-members` |

2. **Admin 权限策略组合**：角色策略 + 范围约束（如只能管理 `customer` 类型的用户）
3. **同一评估引擎**：业务权限和管理权限共享同一评估引擎。FGAP 通过**继承复用**通用评估引擎实现——管理权限评估器在通用评估器基础上扩展，在运行时分发：遇 admin 客户端时路由到管理权限评估器，否则使用通用评估器

这避免了维护两套独立的授权系统。

> **场景示例：** 部门 A 的主管需要能管理本部门的文档，但不能碰 HR 部门的文档：
> ```
> 管理操作 "创建文档" → 映射为资源类型 Clients
> 范围约束 → scope: manage（仅管理权限）+ 属性 filter: department = "A"
> 策略组合 → role(部门主管) + scope(manage) + 属性过滤(仅 department=A)
> → 结果：主管 Alice 可以管理/删除部门 A 的文档，无法操作 HR 文档
> ```

---

## 六、性能与缓存策略

### 6.1 双层缓存

```
请求 → StoreFactory
  │
  ├── CachedStoreFactoryProvider（缓存层）
  │     └── 缓存资源、策略、作用域的读取
  │
  └── JPA Store（持久层）
```

### 6.2 评估缓存（决策缓存）

```
Map<Policy, Map<Object, Effect>> decisionCache

// 第一层 key：策略对象（同一策略的评估结果可复用）
// 第二层 key：视策略类型不同而变化
//   - 资源级策略：key = 资源对象（同资源不同 scope 复用结果）
//   - 作用域/通用策略：key = 权限对象（同权限复用结果）
// 同一请求中避免权限策略树中重复计算相同子策略
```

### 6.3 只读模式

评估过程中设置 `storeFactory.setReadOnly(true)`，防止评估过程意外修改持久化数据，同时允许缓存层做激进优化。

---

## 七、扩展机制

### 9.1 自定义策略类型

```
// 1. 实现策略评估器
class MyPolicyProvider implements PolicyProvider {
    evaluate(evaluation) {
        if (someCondition) {
            evaluation.grant()
        }
    }
}

// 2. 注册策略类型
registerProvider("my-policy", MyPolicyProvider)

// 3. Admin 配置时选择策略类型 "my-policy"
// 4. 运行时评估引擎自动调用
```

### 9.2 自定义评估上下文

`EvaluationContext` 接口提供了请求时的运行时信息，可以扩展注入：

- 网络环境（IP、地理位置）
- 设备信息（浏览器、操作系统）
- 风险评分
- 自定义请求属性

策略评估时可以通过 `evaluation.getContext().getAttributes()` 访问这些信息。

---

## 八、数据流完整示例

### 8.1 已有配置数据

承接零章的文档管理场景，数据库中已存在以下配置：

**resource 表：**

| id | name | type | owner | resource_server_id |
|-----|------|------|-------|-------------------|
| res-1 | `proj-spec-v2` | `project-doc` | dept-A | rs-1 |
| res-2 | `api-design` | `tech-doc` | dept-A | rs-1 |
| res-3 | `employee-reviews` | `hr-doc` | hr-dept | rs-1 |

**scope 表：**

| id | name | resource_server_id |
|-----|------|-------------------|
| scp-read | `read` | rs-1 |
| scp-write | `write` | rs-1 |
| scp-approve | `approve` | rs-1 |
| scp-delete | `delete` | rs-1 |

**resource_scope 表（资源 ↔ 作用域关联）：**

| resource_id | scope_id |
|-------------|----------|
| res-1 | scp-read, scp-write, scp-approve, scp-delete |
| res-2 | scp-read, scp-write |
| res-3 | scp-read, scp-write |

**policy 表：**

| id | name | type | decision_strategy | logic | resource_server_id |
|-----|------|------|-------------------|-------|-------------------|
| pol-1 | `P_project-doc-access` | `resource` | AFFIRMATIVE | POSITIVE | rs-1 |
| pol-2 | `R_editor` | `role` | UNANIMOUS | POSITIVE | rs-1 |
| pol-3 | `PS_read` | `scope` | UNANIMOUS | POSITIVE | rs-1 |

**associated_policy 表（策略嵌套）：**

| policy_id | associated_policy_id |
|-----------|---------------------|
| pol-1 | pol-2 |

**resource_policy 表（策略关联资源）：**

> resource_policy 表支持两种匹配路径：
> - **实例级匹配**：直接存储 `(policy_id, resource_id)` 外键对，命中时精准匹配到具体资源实例
> - **类型级匹配**：通过 `policy_config` 表中的 `defaultResourceType` 键值对与资源的 `type` 字段进行匹配。当 resource_policy 表有 `(pol-1, res-1)` 记录时按实例匹配，无实例匹配时回退到类型匹配

| policy_id | resource_id |
|-----------|-------------|
| pol-1 | res-1（实例级：关联具体项目规格书） |
| pol-2 | —（类型级：通过 config 的 defaultResourceType="tech-doc" 匹配所有技术文档） |

**scope_policy 表（策略关联作用域）：**

| policy_id | scope_id |
|-----------|----------|
| pol-3 | scp-read |

**policy_config 表（策略配置）：**

| policy_id | name | value |
|-----------|------|-------|
| pol-2 | `roles` | `[{"id":"role-editor","required":false}]` |
| pol-2 | `fetchRoles` | `false` |

### 8.2 请求评估过程

评估入口有三条路径：

| 路径 | 触发条件 | 评估方式 |
|------|----------|----------|
| **UMA 路径** | 携带 `permission_ticket`（用户自管理审批票据） | `evaluateUserManagedPermissions()` — 按 ticket 对应的 UMA 策略评估 |
| **通用路径** | 无 ticket，指定具体资源和 scope | `evaluatePermissions()` — 走 resource/scope/类型三层匹配 |
| **全量路径** | 无 ticket，未指定具体权限（请求所有权限） | `evaluateAllPermissions()` — 评估用户全部可用权限 |

以下是通用路径的评估过程：

```
场景：用户 Alice（角色：editor）请求 GET /api/docs/proj-spec-v2
      文档 proj-spec-v2 类型为 "project-doc"，scope 为 "read"
      PolicyEnforcementMode = ENFORCING

Step 1: 确定资源服务器 → 根据请求的 Client 映射到 rs-1
Step 2: 创建 ResourcePermission
          { resource: res-1(proj-spec-v2), scopes: [read] }
Step 3: DefaultPolicyEvaluator.evaluate()
  ├── evaluateResourcePolicies()          ← 按资源实例匹配
  │   └── 查询 resource_policy 表 → 无 pol-1(res-1 不存在实例级关联)
  │       └── 结果：无匹配
  │
  ├── evaluateResourceTypePolicies()      ← 按资源类型匹配
  │   └── 查询 policy_config 表 → config("defaultResourceType"="project-doc")
  │       └── 找到 pol-1 (P_project-doc-access)  ← pol-1 在 resource_policy 中无 res-1 实例关联，但通过类型匹配命中
  │           └── 遍历 associated_policy → 找到 pol-2 (R_editor)
  │               ├── 读取 policy_config: roles=[{id:role-editor}]
  │               └── Alice.hasRole("editor")? → YES → grant
  │
  └── evaluateScopePolicies()
      └── 查询 scope_policy → 找到 pol-3 (PS_read)
          └── 无条件允许 → grant

Step 4: DecisionPermissionCollector 合并结果
  ├── pol-1 → 允许 read（基于资源类型）
  └── pol-3 → 允许 read（基于作用域）
  ├── DecisionStrategy = AFFIRMATIVE → 任一允许即可
  └── 最终：PERMIT（允许 read）

Step 5: 返回 Permission
          { resource: "proj-spec-v2", scopes: ["read"] }
```

### 8.3 拒绝场景走查

```
场景：Alice（editor）请求 DELETE /api/docs/proj-spec-v2

Step 1-2: 创建 ResourcePermission { resource: res-1, scopes: [delete] }

Step 3: 策略评估
  ├── evaluateResourcePolicies() → 无
  ├── evaluateResourceTypePolicies()
  │   └── pol-1 → 委托 R_editor
  │       └── pol-2 的配置只允许 editor 角色的 read/write/approve
  │           └── scope=delete 不在 pol-1 关联的 scope 中 → 不匹配
  └── evaluateScopePolicies()
      └── 无 scope=delete 的策略 → 无匹配

Step 4: ENFORCING 模式下无策略匹配 → 默认拒绝
最终：DENY（拒绝 delete）
```

### 8.4 策略树 + 数据表完整对照

```
Bob（senior-engineer）请求审批 proj-spec-v2：approve
  │
  ├── 匹配 pol-1（通过 config 的 defaultResourceType="project-doc" 类型匹配）
  │     └── resource_policy 表: pol-1 ↔ res-1（实例级关联，优先于类型匹配）
  │     └── policy_config 表: pol-1.defaultResourceType = "project-doc"（类型匹配回退）
  │
  ├── pol-1 委托子策略 pol-2（R_editor）
  │     └── associated_policy 表: pol-1 → pol-2
  │     └── RolePolicyProvider 评估
  │         ├── policy_config: roles=[{id:role-senior-engineer}]
  │         └── Bob.hasRole("senior-engineer")? → YES → grant
  │
  └── 最终：PERMIT（允许 approve）
```

通过对照可以看出：权限配置就是向这些关联表插入数据行，评估就是查询这些表并执行策略逻辑。策略树嵌套的层数对应 `associated_policy` 表中的关联深度。

---

## 九、设计模式总结

| 模式 | 在方案中的体现 |
|------|---------------|
| **组合模式** | 策略嵌套子策略，递归评估 |
| **策略模式** | 通过 type 注册不同的 PolicyProvider |
| **模板方法** | AbstractPermissionProvider 定义递归框架 |
| **责任链** | 多策略按序评估，各自独立判定 |
| **工厂模式** | 按 type 创建 PolicyProvider 实例 |
| **装饰器** | StoreFactory 包装层织入缓存/清理逻辑 |
| **享元模式** | 决策缓存复用同一策略的评估结果 |

---

## 十、设计考量与权衡

### 适用场景
- 需要**多层授权**（角色 + 时间 + 自定义条件）的应用
- 需要**资源所有者自主授权**的共享/协作平台
- 需要**细粒度 admin 分权管理**的企业系统
- 策略规则需要**动态调整**而非硬编码

### 不适合的场景
- 只有简单角色检查的应用 → RBAC 足矣
- 高并发无状态的接口 → 策略评估有开销
- 策略极少变更的应用 → 可以预先编译为权限矩阵

### 复杂度成本
- 学习曲线：理解策略组合、决策策略、作用域需要时间
- 性能开销：策略树递归评估比静态权限表慢
- 存储成本：资源、策略、票证的多表关联查询

---

## 十一、数据表设计

以下是根据 Keycloak 实际 JPA 实体提炼的通用数据表设计，语言无关。

### 11.1 表清单

| 表名 | 说明 |
|------|------|
| `resource_server` | 资源服务器（对应一个应用/微服务） |
| `resource` | 受保护资源定义 |
| `scope` | 作用域定义 |
| `policy` | 策略定义 |
| `permission_ticket` | UMA 权限票据 |
| `resource_attribute` | 资源扩展属性 |
| `resource_uri` | 资源的 URI 集合 |
| `resource_scope` | 资源与作用域的多对多关联 |
| `policy_config` | 策略配置键值对 |
| `associated_policy` | 策略之间的多对多自关联（组合模式） |
| `resource_policy` | 策略与资源的多对多关联 |
| `scope_policy` | 策略与作用域的多对多关联 |

### 11.2 核心表结构

#### resource_server（资源服务器）

| 列 | 类型 | 说明 |
|-----|------|------|
| id | varchar(36) PK | 主键，同时是关联应用的 ID |
| allow_remote_mgmt | boolean | 是否允许远程管理资源 |
| policy_enforce_mode | int | 0=ENFORCING, 1=PERMISSIVE, 2=DISABLED |
| decision_strategy | int | 0=AFFIRMATIVE, 1=UNANIMOUS, 2=CONSENSUS |

#### resource（资源）

| 列 | 类型 | 说明 |
|-----|------|------|
| id | varchar(36) PK | 主键 |
| name | varchar(255) | 资源名称 |
| display_name | varchar(255) | 显示名称 |
| type | varchar(255) | 资源类型（用于按类型批量授权） |
| icon_uri | varchar(255) | 图标 URI |
| owner | varchar(255) | 所有者 ID |
| owner_managed_access | boolean | 是否允许所有者自主授权（UMA） |
| resource_server_id | varchar(36) FK | 所属资源服务器 |

**唯一约束：** `(name, resource_server_id, owner)`

#### scope（作用域）

| 列 | 类型 | 说明 |
|-----|------|------|
| id | varchar(36) PK | 主键 |
| name | varchar(255) | 作用域名称（如 `read`, `write`） |
| display_name | varchar(255) | 显示名称 |
| icon_uri | varchar(255) | 图标 URI |
| resource_server_id | varchar(36) FK NOT NULL | 所属资源服务器 |

**唯一约束：** `(name, resource_server_id)`

#### policy（策略）

| 列 | 类型 | 说明 |
|-----|------|------|
| id | varchar(36) PK | 主键 |
| name | varchar(255) | 策略名称 |
| description | text | 策略描述 |
| type | varchar(255) | 策略类型：`resource`, `scope`, `uma`, `role`, `user`, `group`, `client`, `client-scope`, `time`, `regex`, `aggregate`, `js` 等 |
| decision_strategy | int | 子策略合并策略 |
| logic | int | 0=POSITIVE, 1=NEGATIVE |
| owner | varchar(255) | 所有者（UMA 场景） |
| resource_server_id | varchar(36) FK NOT NULL | 所属资源服务器 |

**唯一约束：** `(name, resource_server_id)`

#### permission_ticket（UMA 权限票据）

| 列 | 类型 | 说明 |
|-----|------|------|
| id | varchar(36) PK | 主键 |
| owner | varchar(255) | 资源所有者 ID |
| requester | varchar(255) | 请求者 ID |
| created_timestamp | bigint | 创建时间 |
| granted_timestamp | bigint | 授权时间（null=未授权） |
| resource_id | varchar(36) FK NOT NULL | 关联资源 |
| scope_id | varchar(36) FK | 关联作用域（null=全部 scope） |
| resource_server_id | varchar(36) FK NOT NULL | 所属资源服务器 |
| policy_id | varchar(36) FK | 授权后关联的策略 |

**唯一约束：** `(owner, requester, resource_server_id, resource_id, scope_id)`

### 11.3 关联表结构

#### resource_scope（资源 ↔ 作用域）

| 列 | 说明 |
|-----|------|
| resource_id FK | 关联资源 |
| scope_id FK | 关联作用域 |

> 关联方向为单向多对多：关联关系定义在资源侧，Scope 表本身不含对资源的反向引用。查询"某 scope 关联了哪些资源"需通过 JOIN 反向查找。

#### resource_policy（资源 ↔ 策略）

| 列 | 说明 |
|-----|------|
| policy_id FK | 关联策略 |
| resource_id FK | 关联资源实例（可为 null，null 时通过 policy_config.defaultResourceType 按类型匹配） |

> **两种匹配路径并存：**
> - **实例级**：`resource_id` 直接指向具体资源 → 评估时通过 JOIN resource_policy 表精准匹配
> - **类型级**：`resource_id` 为空时，通过 `policy_config` 表的 `defaultResourceType` 匹配资源的 `type` 字段 → 实现"对某类型的所有资源生效"的批量授权

#### scope_policy（作用域 ↔ 策略）

| 列 | 说明 |
|-----|------|
| policy_id FK | 关联策略 |
| scope_id FK | 关联作用域 |

#### associated_policy（策略 ↔ 子策略，自关联）

| 列 | 说明 |
|-----|------|
| policy_id FK | 父策略 |
| associated_policy_id FK | 子策略 |

#### policy_config（策略配置）

| 列 | 说明 |
|-----|------|
| policy_id FK | 关联策略 |
| name | 配置键 |
| value | 配置值 |

### 11.4 外键关系总图

```
resource_server ──< resource_server_id ── resource
               ──< resource_server_id ── scope
               ──< resource_server_id ── policy
               ──< resource_server_id ── permission_ticket

resource ──< resource_id ── resource_attribute
resource ──[resource_scope]──> scope
resource ──[resource_policy]<── policy

scope ──[scope_policy]<── policy

policy ──[associated_policy]──> policy (自关联)
policy ──< policy_id ── policy_config

permission_ticket ──< resource_id ── resource
                  ──< scope_id   ── scope
                  ──< policy_id  ── policy
```

### 11.5 设计要点

1. **资源服务器 ID 即应用/客户端 ID**：资源权限系统嵌入在应用内部，不是独立服务
2. **策略自关联实现组合模式**：`associated_policy` 表让策略可以任意深度嵌套
3. **所有资源归 scope 上下文隔离**：每个表都通过 `resource_server_id` 做租户隔离
4. **PermissionTicket 的可选外键**：`scope_id` 和 `policy_id` 可为 null，支持"对整个资源授权"和"未审批状态"

---

## 十二、与 RBAC 模型的结合

资源权限模型**不是替代 RBAC**，而是在 RBAC 之上构建更细粒度的授权层。

### 12.1 整体架构：双轨并行

```
┌─────────────────────────────────────────────┐
│             权限判断入口                       │
│                                              │
│   ┌─────────────────┐  ┌──────────────────┐  │
│   │   RBAC 快速路径   │  │  资源权限评估路径   │  │
│   │  (传统角色检查)    │  │  (Policy Engine)  │  │
│   │                  │  │                   │  │
│   │  hasRole("admin")│  │  ResourcePermission│  │
│   │  → 直接放行      │  │  → Policy Chain   │  │
│   │                  │  │  → 细粒度判定      │  │
│   └────────┬─────────┘  └────────┬──────────┘  │
│            │                     │              │
│            └──────┬──────────────┘              │
│                   ▼                             │
│            ┌──────────────┐                     │
│            │ 任一通过即可  │                     │
│            └──────────────┘                     │
└─────────────────────────────────────────────┘
```

**判断逻辑（故障转移链）：**

```
function hasPermission(user, action, resource):
    // Step 1: RBAC 快速路径
    if user.hasRole("admin") or user.hasRole("manage-xxx"):
        return true
    
    // Step 2: 资源权限引擎评估
    permission = createResourcePermission(resource, action)
    result = policyEvaluator.evaluate(permission, user)
    return result.isGranted()
```

### 12.2 RBAC 角色 → 权限策略：角色策略（Role Policy）

角色策略是连接 RBAC 和资源权限模型的**桥梁**：

```
创建角色策略：
  policy.type = "role"
  policy.config = {
    "roles": [
      { "id": "role-admin-id", "required": true },
      { "id": "role-editor-id", "required": false }
    ],
    "fetchRoles": true   // 是否递归检查角色继承
  }

评估逻辑：
  function evaluate(policy, identity):
    for each roleDef in policy.config.roles:
      hasRole = identity.hasRole(roleDef.id, fetchRoles)
      if roleDef.required and !hasRole:
        return DENY
      if hasRole:
        granted = true
    return granted ? PERMIT : DENY
```

**正向使用：** 将角色策略作为子策略挂在资源/作用域权限策略下

```
资源权限 P (type=resource) → 关联 project-doc → 关联 approve scope
  └── 角色策略 R (type=role) → config: { roles: ["senior-engineer"] }
      → 评估时: Bob.hasRole("senior-engineer")? → 是 → 允许审批
```

**逆向使用：** 从用户的角色反向查找可获得的权限

```
function getPermissionsForUser(user):
    allRoles = getDeepUserRoleMappings(user)  // 含继承和 composite 展开
    // 查找所有包含这些角色的角色策略
    rolePolicies = findPoliciesByRoleIds(allRoles)
    // 通过角色策略向上找到权限策略和资源
    return resolvePermissions(rolePolicies)
```

这实现了"给用户分配一个角色，自动获得对应权限"的效果。

### 12.3 Admin FGAP：RBAC 与资源权限的双轨制

以"管理员能否管理用户 X"为例：

```
Step 1 — RBAC 检查（快速路径）:
  if admin.hasRole("manage-users"):
    return true  // 全局管理员，直接放行

Step 2 — 细粒度检查（资源权限路径）:
  // 构建权限请求
  permission = ResourcePermission {
    resource: UserResource(id=userX),
    scope: "manage"
  }
  // 评估策略链
  //   → 命中角色策略 R（检查 admin 是否有 "department-admin" 角色）
  //   → 命中用户策略 U（检查 admin ID 是否在允许列表中）
  //   → 合并结果
  return policyEvaluator.evaluate(permission, admin)
```

**关键设计：** RBAC 角色（如 `manage-users`）是最粗粒度的授权，覆盖所有资源。资源权限在此基础上做减法——当 RBAC 不授权时，回退到细粒度判断，支持"只管理本部门的用户"这样的场景。

### 12.4 两种模式的适用选择

| 维度 | RBAC 单独使用 | RBAC + 资源权限 |
|------|-------------|----------------|
| 粒度 | 角色级别（所有资源统一） | 资源/作用域级别（每个实例不同） |
| 适用场景 | 简单权限、资源无差异 | 多租户、多类型资源、差异化授权 |
| 性能 | 快（一次角色查询） | 较慢（策略树递归评估） |
| 管理成本 | 低（分配角色即可） | 高（需配置策略） |
| 扩展性 | 新增资源自动继承角色权限 | 新增资源需配置或继承类型策略 |
| 典型组合 | 小团队、单一产品线 | 企业系统、SaaS 平台、文档管理 |

**实际建议：** 绝大多数场景先用 RBAC 覆盖 80% 的权限需求，再对那 20% 需要细粒度控制的资源引入资源权限模型。两套机制共存，通过故障转移链自动降级。

---

> **参考实现：** Keycloak Authorization Services（UMA 2.0 实现）
>
> 核心源码路径：
> - `server-spi-private/src/main/java/org/keycloak/authorization/` — 核心模型与评估引擎
> - `authz/policy/common/src/main/java/org/keycloak/authorization/policy/provider/` — 内置策略实现


## 十三、业务落地实践 — 多租户 SaaS 以新建租户为线索的端到端数据流

本章以「CodeHub」多租户项目管理 SaaS 为背景，以「新建租户」为起点，追踪从组织搭建、角色定义、权限配置到用户访问业务数据的完整数据流转过程。

### 13.1 场景定义

**业务背景：**

「星河科技」是一家新入驻客户（租户），团队包含研发部和产品部。需要为其建立完整的组织架构、角色体系、项目资源和权限控制。

**角色体系：**

| 角色 | 说明 | 菜单权限 |
|------|------|----------|
| `engineer` | 研发工程师 | 项目管理、任务列表、文档库 |
| `pm` | 产品经理 | 项目管理、任务列表、文档库、团队管理 |
| `admin` | 租户管理员 | 全部菜单 |

**用户清单：**

| 用户 | 角色 | 所属部门 |
|------|------|----------|
| Alice | `admin` | 产品部 |
| Bob | `engineer` | 研发部 |
| Charlie | `pm` | 产品部 |

**业务数据：**

| 资源 ID | 名称 | 类型 | 创建人 |
|---------|------|------|--------|
| proj-101 | 星河 CRM | `project` | Charlie |
| proj-102 | 数据中台 | `project` | Alice |
| task-201 | 登录模块 | `task` | Bob |
| task-202 | 数据清洗 | `task` | Bob |
| doc-301 | PRD-v2 | `document` | Charlie |
| doc-302 | 技术方案 | `document` | Bob |

**权限需求：**

| # | 需求 |
|---|------|
| R1 | `engineer` 能查看和编辑所有项目/任务/文档 |
| R2 | `pm` 能查看/编辑/删除项目，能查看/编辑任务和文档 |
| R3 | `admin` 拥有全部资源的所有操作权限 |

**贯穿全章的跟踪变量：**

| 请求 | 用户 | 操作 | 目标资源 | 所需 scope |
|------|------|------|----------|------------|
| Q1 | Bob（engineer） | 查看 | proj-101 | `project:view` |
| Q2 | Bob（engineer） | 删除 | proj-101 | `project:delete` |
| Q3 | Charlie（pm） | 删除 | proj-101 | `project:delete` |

---

### 13.2 数据流：9 步完整推演

以下沿时间线追踪每一步产生的数据库写入，以及 Step 8 中针对 Q1/Q2/Q3 三个请求的完整评估走查。

---

#### Step 1：创建租户 — 写入 resource_server

**业务动作：** 平台管理员在后台为「星河科技」开通 SaaS 工作空间。

**写入数据：**

_resource_server 表_

| id | allow_remote_mgmt | policy_enforce_mode | decision_strategy |
|----|-------------------|---------------------|-------------------|
| `tenant-starry` | false | 0（ENFORCING） | 0（AFFIRMATIVE） |

**说明：** `resource_server` 在此充当租户隔离域的角色。后续所有 resource / scope / policy 均通过 `resource_server_id` 挂在 `tenant-starry` 下，实现租户间数据与策略的物理隔离。

---

#### Step 2：搭建组织架构 — 创建部门与成员

**业务动作：** 管理员 Alice 在「星河科技」下创建研发部和产品部，添加成员。

**操作：** 此步骤不写入授权相关表，但产出 RBAC 基础数据（以下为概念示意，非资源权限模型存储范围）：

- 部门：`dept-eng`（研发部）、`dept-product`（产品部）
- 用户注册：Alice、Bob、Charlie 注册到 `tenant-starry` 域
- 部门成员：Alice ∈ `dept-product`、Bob ∈ `dept-eng`、Charlie ∈ `dept-product`

**说明：** 组织和人员数据存储在 RBAC 层（用户/部门/角色模块），它们不直接写入资源权限表，但为后续角色策略的判定提供依据。当 Step 8 执行 `hasRole()` 检查时，正是查询这里建立的关系。

---

#### Step 3：创建角色 — 写入 role 类型 policy

**业务动作：** Alice 定义三个业务角色。

**Step 3a — 创建角色策略：**

_policy 表_

| id | name | type | decision_strategy | logic | resource_server_id |
|----|------|------|-------------------|-------|-------------------|
| `role-engineer` | 研发工程师 | `role` | UNANIMOUS | POSITIVE | tenant-starry |
| `role-pm` | 产品经理 | `role` | UNANIMOUS | POSITIVE | tenant-starry |
| `role-admin` | 管理员 | `role` | UNANIMOUS | POSITIVE | tenant-starry |

**Step 3b — 写入角色策略的判定配置：**

_policy_config 表_

| policy_id | name | value |
|-----------|------|-------|
| `role-engineer` | `roles` | `[{"id":"role-engineer","required":true}]` |
| `role-engineer` | `fetchRoles` | `false` |
| `role-pm` | `roles` | `[{"id":"role-pm","required":true}]` |
| `role-pm` | `fetchRoles` | `false` |
| `role-admin` | `roles` | `[{"id":"role-admin","required":true}]` |
| `role-admin` | `fetchRoles` | `false` |

**当前进度说明：** 此时角色策略只是三个孤立的判定器——`role-engineer` 知道如何检查用户是否具有 `engineer` 角色，但还没有和任何资源或操作挂钩。后续 Step 6 会将其挂载为子策略。

---

#### Step 4：用户绑定角色 — RBAC 层操作

**业务动作：** Alice 为团队成员分配角色。

**操作：** 在 RBAC 层建立用户-角色关系（概念示意）：

- Alice → `admin`
- Bob → `engineer`
- Charlie → `pm`

**说明：** 角色绑定不在资源权限表中存储，它的价值在 Step 8 体现——评估引擎调用 `identity.hasRole()` 时回溯到这里。

---

#### Step 5：创建业务数据的同时创建 scope — 写入 resource 和 scope

**业务动作：** 团队成员创建项目、任务、文档，同时定义系统中有哪些操作类型。

**Step 5a — 创建 scope（定义操作类型）：**

_scope 表_

| id | name | resource_server_id |
|----|------|-------------------|
| `sc-project-view` | `project:view` | tenant-starry |
| `sc-project-edit` | `project:edit` | tenant-starry |
| `sc-project-del` | `project:delete` | tenant-starry |
| `sc-task-view` | `task:view` | tenant-starry |
| `sc-task-edit` | `task:edit` | tenant-starry |
| `sc-doc-view` | `doc:view` | tenant-starry |
| `sc-doc-edit` | `doc:edit` | tenant-starry |

> scope 定义「能做什么」，按资源类型加操作动词命名，如 `project:view`。

**Step 5b — 创建业务资源并随即挂载 scope：**

_resource 表_

| id | name | type | owner | resource_server_id |
|----|------|------|-------|-------------------|
| `proj-101` | 星河 CRM | `project` | Charlie | tenant-starry |
| `proj-102` | 数据中台 | `project` | Alice | tenant-starry |
| `task-201` | 登录模块 | `task` | Bob | tenant-starry |
| `task-202` | 数据清洗 | `task` | Bob | tenant-starry |
| `doc-301` | PRD-v2 | `document` | Charlie | tenant-starry |
| `doc-302` | 技术方案 | `document` | Bob | tenant-starry |

> 每个资源的 `type` 字段在整个权限体系中起关键作用——它是按类型批量匹配策略的锚点。

**Step 5c — 同时写入 resource_scope（"每个业务数据创建时就挂载上 scope"）：**

_resource_scope 表_

| resource_id | scope_id |
|-------------|----------|
| `proj-101` | `sc-project-view` |
| `proj-101` | `sc-project-edit` |
| `proj-101` | `sc-project-del` |
| `proj-102` | `sc-project-view` |
| `proj-102` | `sc-project-edit` |
| `proj-102` | `sc-project-del` |
| `task-201` | `sc-task-view` |
| `task-201` | `sc-task-edit` |
| `task-202` | `sc-task-view` |
| `task-202` | `sc-task-edit` |
| `doc-301` | `sc-doc-view` |
| `doc-301` | `sc-doc-edit` |
| `doc-302` | `sc-doc-view` |
| `doc-302` | `sc-doc-edit` |

**当前进度说明：** 至此，我们有 3 个角色策略、7 个 scope、6 个资源及它们与 scope 的关联。角色（谁）和 scope（做什么）已经就位，只差把「谁 → 对什么资源 → 能做什么」串起来。这一步在 Step 6 完成。

---

#### Step 6：配置权限策略 — 将角色、资源类型、scope 串联

这是整个流程的**核心步骤**。管理员 Alice 配置三条权限规则：

- R1：`engineer` 角色 → `project`/`task`/`document` 类型资源 → `view`/`edit` scope
- R2：`pm` 角色 → `project` 类型 → `view`/`edit`/`delete`；`task`/`document` 类型 → `view`/`edit`
- R3：`admin` 角色 → 所有类型 → 所有 scope

**Step 6a — 创建 resource 类型权限策略（桥接层）：**

`admin` 角色需要匹配 3 种资源类型，按每种类型一条的原则创建 3 条 admin 策略。

_policy 表_

| id | name | type | decision_strategy | resource_server_id |
|----|------|------|-------------------|-------------------|
| `P-eng-project` | 工程师-项目权限 | `resource` | AFFIRMATIVE | tenant-starry |
| `P-eng-task` | 工程师-任务权限 | `resource` | AFFIRMATIVE | tenant-starry |
| `P-eng-doc` | 工程师-文档权限 | `resource` | AFFIRMATIVE | tenant-starry |
| `P-pm-project` | 产品经理-项目权限 | `resource` | AFFIRMATIVE | tenant-starry |
| `P-pm-task` | 产品经理-任务权限 | `resource` | AFFIRMATIVE | tenant-starry |
| `P-pm-doc` | 产品经理-文档权限 | `resource` | AFFIRMATIVE | tenant-starry |
| `P-admin-project` | 管理员-项目权限 | `resource` | AFFIRMATIVE | tenant-starry |
| `P-admin-task` | 管理员-任务权限 | `resource` | AFFIRMATIVE | tenant-starry |
| `P-admin-doc` | 管理员-文档权限 | `resource` | AFFIRMATIVE | tenant-starry |

**Step 6b — 按资源类型匹配（policy_config）：**

_policy_config 表_

| policy_id | name | value |
|-----------|------|-------|
| `P-eng-project` | `defaultResourceType` | `project` |
| `P-eng-task` | `defaultResourceType` | `task` |
| `P-eng-doc` | `defaultResourceType` | `document` |
| `P-pm-project` | `defaultResourceType` | `project` |
| `P-pm-task` | `defaultResourceType` | `task` |
| `P-pm-doc` | `defaultResourceType` | `document` |
| `P-admin-project` | `defaultResourceType` | `project` |
| `P-admin-task` | `defaultResourceType` | `task` |
| `P-admin-doc` | `defaultResourceType` | `document` |

> 一条 policy 的 `defaultResourceType` 只能存一个值，所以对每种资源类型创建独立策略。`resource_policy` 表此时为空——所有匹配都走类型匹配。

**Step 6c — 绑定 scope（scope_policy）：**

_scope_policy 表_

| policy_id | scope_id |
|-----------|----------|
| `P-eng-project` | `sc-project-view` |
| `P-eng-project` | `sc-project-edit` |
| `P-eng-task` | `sc-task-view` |
| `P-eng-task` | `sc-task-edit` |
| `P-eng-doc` | `sc-doc-view` |
| `P-eng-doc` | `sc-doc-edit` |
| `P-pm-project` | `sc-project-view` |
| `P-pm-project` | `sc-project-edit` |
| `P-pm-project` | `sc-project-del` |
| `P-pm-task` | `sc-task-view` |
| `P-pm-task` | `sc-task-edit` |
| `P-pm-doc` | `sc-doc-view` |
| `P-pm-doc` | `sc-doc-edit` |
| `P-admin-project` | `sc-project-view` |
| `P-admin-project` | `sc-project-edit` |
| `P-admin-project` | `sc-project-del` |
| `P-admin-task` | `sc-task-view` |
| `P-admin-task` | `sc-task-edit` |
| `P-admin-doc` | `sc-doc-view` |
| `P-admin-doc` | `sc-doc-edit` |

> 这一步绑定了「resource 类型策略能覆盖哪些操作」。例如 `P-eng-project` 关联 `sc-project-view` 和 `sc-project-edit` 两个 scope——工程师可以查看和编辑项目，但不能删除。

**Step 6d — 挂载角色策略（associated_policy）：**

_associated_policy 表_

| policy_id | associated_policy_id |
|-----------|---------------------|
| `P-eng-project` | `role-engineer` |
| `P-eng-task` | `role-engineer` |
| `P-eng-doc` | `role-engineer` |
| `P-pm-project` | `role-pm` |
| `P-pm-task` | `role-pm` |
| `P-pm-doc` | `role-pm` |
| `P-admin-project` | `role-admin` |
| `P-admin-task` | `role-admin` |
| `P-admin-doc` | `role-admin` |

**Step 6 完成后，策略树全貌：**

```
P-eng-project  (resource, scope=[view,edit], type=project)
  └── role-engineer  (检查 identity.hasRole("engineer"))

P-eng-task  (resource, scope=[view,edit], type=task)
  └── role-engineer

P-eng-doc  (resource, scope=[view,edit], type=document)
  └── role-engineer

P-pm-project  (resource, scope=[view,edit,delete], type=project)
  └── role-pm  (检查 identity.hasRole("pm"))

P-pm-task  (resource, scope=[view,edit], type=task)
  └── role-pm

P-pm-doc  (resource, scope=[view,edit], type=document)
  └── role-pm

P-admin-project  (resource, scope=[view,edit,delete], type=project)
  └── role-admin  (检查 identity.hasRole("admin"))

P-admin-task  (resource, scope=[view,edit], type=task)
  └── role-admin

P-admin-doc  (resource, scope=[view,edit], type=document)
  └── role-admin
```

**resource_policy 表当前状态：空**

本案例中所有权限策略均通过 `defaultResourceType` 类型匹配，无实例级策略。

---

#### Step 7：用户发起业务请求

**业务动作：** 用户在前端页面操作业务数据。

| 请求 | 用户 | 操作 | 目标资源 | 所需 scope | 期望结果 |
|------|------|------|----------|------------|----------|
| Q1 | Bob（engineer） | 查看 | proj-101 | `project:view` | 允许 |
| Q2 | Bob（engineer） | 删除 | proj-101 | `project:delete` | 拒绝 |
| Q3 | Charlie（pm） | 删除 | proj-101 | `project:delete` | 允许 |

---

#### Step 8：权限判定 — 三个请求的完整数据流转

##### Q1：Bob（engineer）查看 proj-101

```
=== 阶段 1：定位资源服务器 ===
从请求上下文获取 resource_server → tenant-starry
检查 policy_enforce_mode → ENFORCING

=== 阶段 2：构造 ResourcePermission ===
resource = proj-101 (type=project)
scope   = project:view (sc-project-view)

=== 阶段 3：查询匹配策略 ===

3a. 按资源实例匹配（resource_policy 表）:
    查询: SELECT * FROM resource_policy WHERE resource_id = 'proj-101'
    → 结果: 空（没有为 proj-101 配置实例级策略）

3b. 按资源类型匹配（policy_config 表）:
    查询: policy_config WHERE KEY = 'defaultResourceType' AND value = 'project'
          JOIN policy ON p.resource_server_id = 'tenant-starry'
    → 命中: P-eng-project, P-pm-project, P-admin-project（3条）

3c. 按 scope 匹配（scope_policy 表）:
    在上一步命中的 3 条中筛选 scope_id = 'sc-project-view'：

    P-eng-project:
      SELECT * FROM scope_policy
      WHERE policy_id = 'P-eng-project' AND scope_id = 'sc-project-view'
      → 命中 ✓

    P-pm-project:
      SELECT * FROM scope_policy
      WHERE policy_id = 'P-pm-project' AND scope_id = 'sc-project-view'
      → 命中 ✓

    P-admin-project:
      SELECT * FROM scope_policy
      WHERE policy_id = 'P-admin-project' AND scope_id = 'sc-project-view'
      → 命中 ✓

    → 最终命中策略: P-eng-project, P-pm-project, P-admin-project（3条同时满足类型+scope匹配）

=== 阶段 4：递归评估策略树 ===

    评估 P-eng-project:
      加载子策略: associated_policy WHERE policy_id = 'P-eng-project'
      → role-engineer
      读取子策略配置: policy_config WHERE policy_id = 'role-engineer'
      → roles=[{"id":"role-engineer","required":true}]
      判定: Bob.hasRole("engineer")? → 是 → GRANT
      → P-eng-project = GRANT

    评估 P-pm-project:
      加载子策略: role-pm
      配置: roles=[{"id":"role-pm","required":true}]
      判定: Bob.hasRole("pm")? → 否 → 不产生 effect
      → denyIfNoEffect() → P-pm-project = DENY

    评估 P-admin-project:
      加载子策略: role-admin
      配置: roles=[{"id":"role-admin","required":true}]
      判定: Bob.hasRole("admin")? → 否 → 不产生 effect
      → denyIfNoEffect() → P-admin-project = DENY

=== 阶段 5：合并结果 ===
resource_server.decision_strategy = AFFIRMATIVE
→ 任一条 GRANT → 最终 GRANT

P-eng-project = GRANT
→ 最终: PERMIT（允许查看）✓
```

##### Q2：Bob（engineer）删除 proj-101

```
阶段 1-2: 同 Q1，scope 改为 project:delete (sc-project-del)

阶段 3: 查询匹配策略

3a. 按资源实例: 空（同上）

3b. 按资源类型: 命中 P-eng-project, P-pm-project, P-admin-project（同上）

3c. 按 scope 筛选:

    P-eng-project: scope_policy 中没有 sc-project-del → 不匹配 ✗
    P-pm-project:  scope_policy 中有 sc-project-del    → 匹配 ✓
    P-admin-project: scope_policy 中有 sc-project-del   → 匹配 ✓

    → 命中策略: P-pm-project, P-admin-project（2条）

阶段 4: 评估

    评估 P-pm-project:
      → role-pm → Bob.hasRole("pm")? → 否 → denyIfNoEffect → DENY

    评估 P-admin-project:
      → role-admin → Bob.hasRole("admin")? → 否 → denyIfNoEffect → DENY

阶段 5: AFFIRMATIVE → 全部 DENY → 最终 DENY
→ Bob 不能删除 proj-101 ✓
```

##### Q3：Charlie（pm）删除 proj-101

```
阶段 1-3: 同 Q2，命中 P-pm-project, P-admin-project（2条）

阶段 4: 评估

    评估 P-pm-project:
      → role-pm → Charlie.hasRole("pm")? → 是 → GRANT
      → P-pm-project = GRANT

    评估 P-admin-project:
      → role-admin → Charlie.hasRole("admin")? → 否 → denyIfNoEffect → DENY

阶段 5: AFFIRMATIVE → P-pm-project 已 GRANT → 最终 GRANT
→ Charlie 可以删除 proj-101 ✓
```

---

#### Step 9：数据膨胀观察 — 新增业务资源触发的写操作

新增一个项目「移动端 App」（`proj-103`, 创建人 Alice, type=`project`）后：

**需要写入的数据：**

```
resource:        INSERT 1 行  (proj-103, project, Alice, tenant-starry)
resource_scope:  INSERT 3 行  (proj-103 ↔ sc-project-view/edit/del)
```

**无需变更的数据：**

```
policy:            不变（P-eng-project/P-pm-project/P-admin-project 已存在）
policy_config:     不变（defaultResourceType = 'project' 已配置）
associated_policy: 不变（role-engineer/role-pm/role-admin 已挂载）
scope_policy:      不变（project:view/edit/del 已绑定）
resource_policy:   不变（仍通过类型匹配）
```

**新增项目前后各表行数对比：**

| 表 | 新增前 | 新增后 | +Δ |
|----|--------|--------|-----|
| resource | 6 | 7 | +1 |
| resource_scope | 14 | 17 | +3 |
| policy | 12 | 12 | 0 |
| policy_config | 15 | 15 | 0 |
| associated_policy | 9 | 9 | 0 |
| scope_policy | 20 | 20 | 0 |

新增一个任务「推送通知」（`task-203`）同理，仅需在 resource 和 resource_scope 插入。无论多少项目/任务/文档，策略层规模保持恒定。

---

### 13.3 数据表全貌快照

以下汇总 9 步流程完成后，所有授权相关表的完整数据行。

#### resource_server（1 行）

| id | allow_remote_mgmt | policy_enforce_mode | decision_strategy |
|----|-------------------|---------------------|-------------------|
| `tenant-starry` | false | 0（ENFORCING） | 0（AFFIRMATIVE） |

#### resource（6 行）

| id | name | type | owner | owner_managed_access | resource_server_id |
|----|------|------|-------|---------------------|-------------------|
| `proj-101` | 星河 CRM | `project` | Charlie | false | tenant-starry |
| `proj-102` | 数据中台 | `project` | Alice | false | tenant-starry |
| `task-201` | 登录模块 | `task` | Bob | false | tenant-starry |
| `task-202` | 数据清洗 | `task` | Bob | false | tenant-starry |
| `doc-301` | PRD-v2 | `document` | Charlie | false | tenant-starry |
| `doc-302` | 技术方案 | `document` | Bob | false | tenant-starry |

#### scope（7 行）

| id | name | resource_server_id |
|----|------|-------------------|
| `sc-project-view` | `project:view` | tenant-starry |
| `sc-project-edit` | `project:edit` | tenant-starry |
| `sc-project-del` | `project:delete` | tenant-starry |
| `sc-task-view` | `task:view` | tenant-starry |
| `sc-task-edit` | `task:edit` | tenant-starry |
| `sc-doc-view` | `doc:view` | tenant-starry |
| `sc-doc-edit` | `doc:edit` | tenant-starry |

#### resource_scope（14 行）

| resource_id | scope_id | | resource_id | scope_id |
|-------------|----------|---|-------------|----------|
| `proj-101` | `sc-project-view` | | `task-202` | `sc-task-view` |
| `proj-101` | `sc-project-edit` | | `task-202` | `sc-task-edit` |
| `proj-101` | `sc-project-del` | | `doc-301` | `sc-doc-view` |
| `proj-102` | `sc-project-view` | | `doc-301` | `sc-doc-edit` |
| `proj-102` | `sc-project-edit` | | `doc-302` | `sc-doc-view` |
| `proj-102` | `sc-project-del` | | `doc-302` | `sc-doc-edit` |
| `task-201` | `sc-task-view` | | | |
| `task-201` | `sc-task-edit` | | | |

#### policy（12 行）

| id | name | type | decision_strategy | logic | resource_server_id |
|----|------|------|-------------------|-------|-------------------|
| `role-engineer` | 研发工程师 | `role` | UNANIMOUS | POSITIVE | tenant-starry |
| `role-pm` | 产品经理 | `role` | UNANIMOUS | POSITIVE | tenant-starry |
| `role-admin` | 管理员 | `role` | UNANIMOUS | POSITIVE | tenant-starry |
| `P-eng-project` | 工程师-项目权限 | `resource` | AFFIRMATIVE | POSITIVE | tenant-starry |
| `P-eng-task` | 工程师-任务权限 | `resource` | AFFIRMATIVE | POSITIVE | tenant-starry |
| `P-eng-doc` | 工程师-文档权限 | `resource` | AFFIRMATIVE | POSITIVE | tenant-starry |
| `P-pm-project` | 产品经理-项目权限 | `resource` | AFFIRMATIVE | POSITIVE | tenant-starry |
| `P-pm-task` | 产品经理-任务权限 | `resource` | AFFIRMATIVE | POSITIVE | tenant-starry |
| `P-pm-doc` | 产品经理-文档权限 | `resource` | AFFIRMATIVE | POSITIVE | tenant-starry |
| `P-admin-project` | 管理员-项目权限 | `resource` | AFFIRMATIVE | POSITIVE | tenant-starry |
| `P-admin-task` | 管理员-任务权限 | `resource` | AFFIRMATIVE | POSITIVE | tenant-starry |
| `P-admin-doc` | 管理员-文档权限 | `resource` | AFFIRMATIVE | POSITIVE | tenant-starry |

#### policy_config（15 行）

| policy_id | name | value |
|-----------|------|-------|
| `role-engineer` | `roles` | `[{"id":"role-engineer","required":true}]` |
| `role-engineer` | `fetchRoles` | `false` |
| `role-pm` | `roles` | `[{"id":"role-pm","required":true}]` |
| `role-pm` | `fetchRoles` | `false` |
| `role-admin` | `roles` | `[{"id":"role-admin","required":true}]` |
| `role-admin` | `fetchRoles` | `false` |
| `P-eng-project` | `defaultResourceType` | `project` |
| `P-eng-task` | `defaultResourceType` | `task` |
| `P-eng-doc` | `defaultResourceType` | `document` |
| `P-pm-project` | `defaultResourceType` | `project` |
| `P-pm-task` | `defaultResourceType` | `task` |
| `P-pm-doc` | `defaultResourceType` | `document` |
| `P-admin-project` | `defaultResourceType` | `project` |
| `P-admin-task` | `defaultResourceType` | `task` |
| `P-admin-doc` | `defaultResourceType` | `document` |

#### associated_policy（9 行）

| policy_id | associated_policy_id |
|-----------|---------------------|
| `P-eng-project` | `role-engineer` |
| `P-eng-task` | `role-engineer` |
| `P-eng-doc` | `role-engineer` |
| `P-pm-project` | `role-pm` |
| `P-pm-task` | `role-pm` |
| `P-pm-doc` | `role-pm` |
| `P-admin-project` | `role-admin` |
| `P-admin-task` | `role-admin` |
| `P-admin-doc` | `role-admin` |

#### scope_policy（20 行）

| policy_id | scope_id |
|-----------|----------|
| `P-eng-project` | `sc-project-view` |
| `P-eng-project` | `sc-project-edit` |
| `P-eng-task` | `sc-task-view` |
| `P-eng-task` | `sc-task-edit` |
| `P-eng-doc` | `sc-doc-view` |
| `P-eng-doc` | `sc-doc-edit` |
| `P-pm-project` | `sc-project-view` |
| `P-pm-project` | `sc-project-edit` |
| `P-pm-project` | `sc-project-del` |
| `P-pm-task` | `sc-task-view` |
| `P-pm-task` | `sc-task-edit` |
| `P-pm-doc` | `sc-doc-view` |
| `P-pm-doc` | `sc-doc-edit` |
| `P-admin-project` | `sc-project-view` |
| `P-admin-project` | `sc-project-edit` |
| `P-admin-project` | `sc-project-del` |
| `P-admin-task` | `sc-task-view` |
| `P-admin-task` | `sc-task-edit` |
| `P-admin-doc` | `sc-doc-view` |
| `P-admin-doc` | `sc-doc-edit` |

#### resource_policy（0 行）

本案例中所有策略均通过类型匹配，此表为空。

---

### 13.4 设计总结

| 业务关注点 | 在模型中的体现 | 数据行变化 |
|-----------|---------------|-----------|
| 新建租户 | 插入一行 `resource_server`，全部资源/策略以 `resource_server_id` 隔离 | +1 行 |
| 新增角色 | 创建一条 `type=role` 的 policy + 对应的 policy_config | +1 policy, +1~2 config |
| 角色绑定菜单权限 | `role` 策略 + `resource` 策略 + `scope_policy` 三层关联 | 组合后约 3 行/menu |
| 用户绑定角色 | RBAC 层操作，评估时 `hasRole()` 回溯 | 不写入本章表 |
| 新增业务数据 | 插入 `resource` + `resource_scope`，策略通过类型匹配自动覆盖 | +1 resource, +N scope |
| 数据膨胀控制 | 策略层规模与业务数据量解耦，`resource_policy` 仅用于实例级例外 | 策略表行数恒定 |

**数据增长曲线（本案例 6 个资源规模）：**

```
表              行数    增长模式
────────────────────────────────
resource_server   1     每租户 +1
scope             7     每种操作 +1，与资源数量无关
resource          6     每业务对象 +1
resource_scope   14     resource × 关联 scope 数
policy           12     角色数 + 角色数×资源类型数
policy_config    15     与 policy 成正比
associated_policy 9     角色数×资源类型数
scope_policy     20     resource型policy × 关联scope数
resource_policy   0     仅实例级例外时产生
```

当资源从 6 个增长到 1000 个时，策略相关的 5 张表（policy / policy_config / associated_policy / scope_policy / resource_policy）行数不变，只有 resource 和 resource_scope 线性增长。
