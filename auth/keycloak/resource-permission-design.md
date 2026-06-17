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
    ├── role:    检查角色（支持角色继承）
    ├── user:    检查用户 ID
    ├── group:   检查组归属
    ├── client:  检查客户端 ID
    ├── time:    检查时间范围
    ├── js:      自定义脚本（最灵活）
    └── ...      可 SPI 扩展
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
  │     ├── 按资源 R 查找 → 找到权限策略 P1
  │     ├── 按资源类型查找 → 找到权限策略 P2
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

1. **管理操作映射为资源类型**：`UserResource`, `RoleResource`, `GroupResource`, `ClientResource`
2. **Admin 权限策略组合**：角色策略 + 范围约束（如只能管理 `customer` 类型的用户）
3. **同一评估引擎**：业务权限和管理权限共享 `DefaultPolicyEvaluator`

这避免了维护两套独立的授权系统。

> **场景示例：** 部门 A 的主管需要能管理本部门的文档，但不能碰 HR 部门的文档：
> ```
> 管理操作 "创建文档" → 映射为资源类型 UserResource
> 范围约束 → 资源 filter: department = "A"
> 策略组合 → role(部门主管) + scope(create/delete) + scope(仅 department=A)
> → 结果：主管 Alice 可以创建/删除部门 A 的文档，无法操作 HR 文档
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
Map<Policy, Map<Permission, Effect>> decisionCache

// 同一请求中，同一策略 + 同一权限的评估结果直接复用
// 避免权限策略树中重复计算相同子策略
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

**resource_policy 表（策略关联资源类型）：**

| policy_id | resource_id |
|-----------|-------------|
| pol-1 | (关联 type=project-doc，按类型匹配，非具体资源 ID) |

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

```
场景：用户 Alice（角色：editor）请求 GET /api/docs/proj-spec-v2
      文档 proj-spec-v2 类型为 "project-doc"，scope 为 "read"
      PolicyEnforcementMode = ENFORCING

Step 1: 确定资源服务器 → 根据请求的 Client 映射到 rs-1
Step 2: 创建 ResourcePermission
          { resource: res-1(proj-spec-v2), scopes: [read] }
Step 3: DefaultPolicyEvaluator.evaluate()
  ├── evaluateResourcePolicies()
  │   └── 查询 policy → resource_policy → 无 res-1 专属策略
  │       └── 结果：无匹配
  │
  ├── evaluateResourceTypePolicies()
  │   └── 查询 policy → config("defaultResourceType"="project-doc")
  │       └── 找到 pol-1 (P_project-doc-access)
  │           └── 委托 AbstractPermissionProvider
  │               └── 遍历 associated_policy → 找到 pol-2 (R_editor)
  │                   └── 委托 RolePolicyProvider
  │                       ├── 读取 policy_config: roles=[{id:role-editor}]
  │                       └── Alice.hasRole("editor")? → YES → grant
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
  ├── 匹配 pol-1（关联 type=project-doc, scope=approve）
  │     └── resource_policy 表: pol-1 ↔ project-doc
  │     └── policy_config 表: pol-1 的 config 无特殊配置
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
| type | varchar(255) | 策略类型：`resource`, `scope`, `uma`, `role`, `user`, `js`, `time` 等 |
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

#### resource_policy（资源 ↔ 策略）

| 列 | 说明 |
|-----|------|
| policy_id FK | 关联策略 |
| resource_id FK | 关联资源 |

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
