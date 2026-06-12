# Leros 数据库设计

## 1. 规范说明

- **ORM**: GORM v1.30
- **支持的数据库**: SQLite / PostgreSQL / MySQL / ClickHouse
- **表名前缀**: 统一使用 `leros_`
- **ID 生成**: 使用 `gorm.Model` 内嵌，自增主键（类型 `uint`）
- **公共字段**（`gorm.Model` 嵌入）:
  - `id` — 自增主键
  - `created_at` — 创建时间
  - `updated_at` — 更新时间
  - `deleted_at` — 软删除标记（唯一索引中需注意 `WHERE` 条件过滤已删除记录）

---

## 2. 表定义

### 2.1 组织与用户

#### leros_organization — 组织/租户

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | uint | PK | |
| `created_at` | timestamp | | |
| `updated_at` | timestamp | | |
| `deleted_at` | timestamp | nullable | 软删除 |
| `public_id` | varchar(64) | UNIQUE, NOT NULL | 对外公开标识 |
| `type` | varchar(50) | DEFAULT `company` | 组织类型 |
| `code` | varchar(255) | UNIQUE, NOT NULL | 组织编码（业务唯一） |
| `name` | varchar(255) | NOT NULL | 组织名称 |
| `status` | varchar(20) | DEFAULT `active` | 状态 |

**用例**: 多租户隔离，一个组织包含多个用户、项目、数字助手。

---

#### leros_user — 用户

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | uint | PK | |
| `public_id` | varchar(64) | UNIQUE, NOT NULL, DEFAULT `''` | 对外公开标识 |
| `github_id` | bigint | UNIQUE | GitHub 用户 ID |
| `github_login` | varchar(255) | UNIQUE, NOT NULL | GitHub 登录名 |
| `password` | varchar(255) | nullable | 密码（可选，本地认证用） |
| `name` | varchar(255) | | 姓名 |
| `email` | varchar(255) | | 邮箱 |
| `avatar_url` | varchar(500) | | 头像 |
| `bio` | text | | 简介 |
| `company` | varchar(255) | | 公司 |
| `location` | varchar(255) | | 位置 |
| `public_repos` | integer | | 公开仓库数 |
| `followers` | integer | | 关注者数 |

**用例**: 支持 GitHub OAuth 和本地密码两种认证方式。

---

#### leros_user_org — 用户-组织关联

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | uint | PK | |
| `uin` | bigint | UNIQUE, NOT NULL | 关联 ID（JWT 中的 Uin） |
| `user_id` | bigint | NOT NULL, INDEX | 用户 ID |
| `org_id` | bigint | NOT NULL, INDEX | 组织 ID |
| `is_default` | boolean | DEFAULT `false` | 是否为用户的默认组织 |

**用例**: 用户可同时归属于多个组织，`is_default` 标识默认进入的组织。

---

### 2.2 认证

#### leros_auth_refresh_token — 刷新令牌

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | uint | PK | |
| `token_hash` | varchar(64) | UNIQUE, NOT NULL | Token 哈希值（不存明文） |
| `user_id` | bigint | NOT NULL, INDEX | 用户 ID |
| `expires_at` | timestamp | NOT NULL, INDEX | 过期时间 |
| `revoked_at` | timestamp | nullable, INDEX | 撤销时间 |

**用例**: 支持 Token 刷新轮换，撤销后不可使用。

---

#### leros_auth_login_attempt — 登录尝试记录

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | uint | PK | |
| `identifier` | varchar(255) | UNIQUE, NOT NULL | 登录标识（邮箱/用户名） |
| `failure_count` | integer | NOT NULL, DEFAULT `0` | 失败次数 |
| `window_expires_at` | timestamp | NOT NULL, INDEX | 窗口过期时间（超时重置） |

**用例**: 登录限流，防止暴力破解。

---

### 2.3 数字助手

#### leros_digital_assistant — AI 数字员工

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | uint | PK | |
| `code` | varchar(255) | NOT NULL, INDEX | 助手标识符（组织内唯一） |
| `org_id` | integer | NOT NULL, INDEX | 所属组织 ID |
| `owner_id` | integer | NOT NULL, INDEX | 拥有者用户 ID |
| `name` | varchar(255) | NOT NULL | 名称 |
| `description` | text | | 描述 |
| `avatar` | varchar(500) | | 头像 URL |
| `status` | varchar(50) | NOT NULL | 状态 |
| `version` | integer | DEFAULT `0` | 版本号 |
| `system_prompt` | text | | 系统提示词 |

**用例**: 数字员工是 AI 工作者的顶层抽象，一个数字员工可被分配给多个会话/项目。

---

### 2.4 项目管理

#### leros_project — 项目

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | uint | PK | |
| `public_id` | varchar(255) | UNIQUE, NOT NULL | 对外公开标识 |
| `org_id` | integer | NOT NULL, INDEX | 所属组织 ID |
| `owner_id` | integer | NOT NULL, INDEX | 创建者 ID |
| `name` | varchar(255) | NOT NULL | 项目名称 |
| `description` | text | | 项目描述 |
| `objective` | text | | 项目目标 |
| `status` | varchar(50) | NOT NULL, DEFAULT `active`, INDEX | 状态 |
| `metadata` | jsonb | | 元数据（ObjectMetadata） |

**索引**:
- `public_id` 唯一索引
- `org_id` 索引
- `status` 索引

**用例**: 项目的核心容器，聚合任务、会话、成员和产物。

---

#### leros_project_member — 项目成员

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | uint | PK | |
| `project_id` | bigint | NOT NULL, INDEX | 项目 ID |
| `member_id` | bigint | NOT NULL, INDEX | 成员 ID |
| `member_type` | varchar(50) | NOT NULL, DEFAULT `user` | 成员类型（user / assistant） |
| `member_role` | varchar(50) | NOT NULL, DEFAULT `member` | 成员角色（owner / admin / member / viewer） |
| `joined_at` | timestamp | NOT NULL, DEFAULT now() | 加入时间 |

**用例**: 项目和成员的多对多关联，成员可以是用户或数字助手。

---

### 2.5 任务

#### leros_task — 任务

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | uint | PK | |
| `public_id` | varchar(255) | UNIQUE, NOT NULL | 对外公开标识 |
| `org_id` | integer | NOT NULL, INDEX | 所属组织 ID |
| `owner_id` | integer | NOT NULL, INDEX | 创建者 ID |
| `project_id` | bigint | NOT NULL, INDEX | 所属项目 ID |
| `session_id` | bigint | nullable, INDEX | 关联会话 ID |
| `task_type` | varchar(16) | NOT NULL, DEFAULT `general`, INDEX | 任务类型（general / cron） |
| `assignee_id` | bigint | nullable, INDEX | 被分配者 ID |
| `title` | varchar(500) | NOT NULL | 任务标题 |
| `description` | text | | 任务描述 |
| `status` | varchar(50) | NOT NULL, DEFAULT `created`, INDEX | 状态 |
| `deadline` | timestamp | nullable, INDEX | 截止日期 |
| `metadata` | jsonb | | 元数据（ObjectMetadata） |

**状态流转**: `created` → `pending_assignment` → `in_progress` → `pending_approval` → `completed` / `failed` / `cancelled` / `paused`

**用例**: 最小执行单元，一个项目下可包含多个任务。

---

### 2.6 会话与消息

#### leros_session — 会话

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | uint | PK | |
| `public_id` | varchar(255) | UNIQUE, NOT NULL | 对外公开标识 |
| `type` | varchar(50) | NOT NULL | 会话类型 |
| `uin` | bigint | DEFAULT `0`, INDEX | 关联用户 UIN |
| `org_id` | integer | NOT NULL, DEFAULT `0`, INDEX | 所属组织 ID |
| `assistant_id` | bigint | DEFAULT `0`, INDEX | 关联数字助手 ID |
| `allocated_assistant_id` | bigint | DEFAULT `0`, INDEX | 分配的数字员工 ID |
| `task_id` | bigint | nullable | 关联任务 ID |
| `project_id` | bigint | nullable | 关联项目 ID |
| `status` | varchar(50) | NOT NULL, DEFAULT `active` | 状态 |
| `title` | varchar(500) | | 会话标题 |
| `title_manually_set` | boolean | NOT NULL, DEFAULT `false` | 用户是否手动修改了标题 |
| `metadata` | jsonb | | 元数据（标签、类型等） |
| `message_count` | integer | DEFAULT `0` | 消息数量 |
| `last_message_at` | timestamp | nullable | 最后消息时间 |
| `expired_at` | timestamp | nullable, INDEX | 过期时间 |

**索引**:
- `public_id` 唯一索引
- `(task_id, project_id)` 联合唯一索引 `uni_session_project_task`（允许 NULL）

**说明**: 会话有三种作用域——归属于项目（project）、归属于任务（task）、独立会话（standalone）。`task_id` 和 `project_id` 同时不为空时通过联合唯一索引约束。

---

#### leros_session_message — 会话消息

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | uint | PK | |
| `session_id` | bigint | NOT NULL, INDEX | 所属会话 ID |
| `role` | varchar(50) | NOT NULL | 消息角色 |
| `content` | text | NOT NULL | 消息内容 |
| `message_type` | varchar(50) | DEFAULT `text` | 消息类型 |
| `status` | varchar(50) | DEFAULT `complete` | 消息状态（complete / streaming） |
| `chunks` | jsonb | | 流式片段（MessageChunkSlice） |
| `artifacts` | jsonb | | 关联产物引用（MessageArtifactSlice） |
| `metadata` | jsonb | | 消息元数据（ObjectMetadata） |
| `usage` | jsonb | | Token 用量（MessageUsage: input_tokens, output_tokens, total_tokens） |
| `sequence` | bigint | NOT NULL, INDEX | 消息序号 |
| `timestamp` | bigint | | 时间戳（Unix 毫秒） |

**索引**:
- `session_id` 索引
- `sequence` 索引（同 session 内按序号排序）

**JSONB 子结构**:

```jsonc
// chunks（流式片段）
[
  {
    "seq": 1,
    "last_seq": 10,
    "type": "text",
    "timestamp": 1718000000000,
    "payload": { /* json.RawMessage */ }
  }
]

// artifacts（产物引用）
[
  {
    "artifact_id": 1,
    "title": "report.pdf",
    "filename": "report.pdf",
    "mime_type": "application/pdf",
    "artifact_type": "file"
  }
]

// usage（Token 用量）
{
  "input_tokens": 1500,
  "output_tokens": 800,
  "total_tokens": 2300
}
```

**用例**: 消息支持流式推送（chunks）、产物引用（artifacts），`sequence` 保证顺序。

---

### 2.7 产物

#### leros_artifact — 任务产出物

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | uint | PK | |
| `public_id` | varchar(255) | UNIQUE, NOT NULL | 对外公开标识 |
| `org_id` | integer | NOT NULL, INDEX | 所属组织 ID |
| `owner_id` | bigint | NOT NULL, INDEX | 拥有者 ID |
| `task_id` | bigint | NOT NULL, INDEX | 关联任务 ID |
| `project_id` | bigint | NOT NULL, INDEX | 关联项目 ID（冗余字段） |
| `session_id` | bigint | nullable, INDEX | 关联会话 ID |
| `message_id` | bigint | nullable, INDEX | 关联消息 ID |
| `title` | varchar(500) | NOT NULL | 标题 |
| `filename` | varchar(500) | | 文件名 |
| `description` | text | | 描述 |
| `artifact_type` | varchar(50) | NOT NULL, INDEX | 产物类型（text / structured_data / file / composite） |
| `file_url` | varchar(2000) | NOT NULL | 文件路径/链接 |
| `mime_type` | varchar(100) | | MIME 类型 |
| `file_size` | bigint | | 文件大小 |
| `relative_path` | varchar(2000) | | 相对路径 |
| `storage_key` | varchar(2000) | INDEX | 存储键 |
| `sha256` | varchar(64) | INDEX | 文件哈希 |
| `source` | varchar(50) | NOT NULL, DEFAULT `agent_declared`, INDEX | 来源（agent_declared / diff） |
| `export_format` | varchar(50) | | 导出格式 |
| `version` | integer | NOT NULL, DEFAULT `1` | 版本号 |
| `parent_id` | bigint | nullable | 父版本 ID（用于版本链） |
| `status` | varchar(50) | NOT NULL, DEFAULT `completed`, INDEX | 状态（generating / completed / failed） |
| `metadata` | jsonb | | 元数据（ObjectMetadata） |

**用例**: 产物支持版本管理（`version` + `parent_id`），可关联到任务、会话和消息。

---

### 2.8 LLM 模型

#### leros_llm_model — LLM 模型配置

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | uint | PK | |
| `org_id` | integer | NOT NULL, INDEX | 所属组织 ID |
| `code` | varchar(128) | NOT NULL | 配置编码 |
| `name` | varchar(255) | NOT NULL | 展示名称 |
| `description` | text | | 描述 |
| `provider` | varchar(64) | NOT NULL, INDEX | 供应商（openai / anthropic / deepseek / qwen / gemini / ark / openrouter / custom） |
| `model` | varchar(255) | NOT NULL | 实际模型名称（如 gpt-4、claude-3） |
| `base_url` | varchar(500) | | API 基础地址 |
| `base_url_has_v1` | boolean | DEFAULT `true` | 基础地址是否含 `/v1` |
| `api_key_encrypted` | text | | API Key 密文 |
| `api_key_masked` | varchar(128) | | API Key 脱敏值 |
| `max_tokens` | integer | DEFAULT `4096` | 默认最大输出 Token 数 |
| `temperature` | decimal(4,3) | DEFAULT `0.7` | 默认采样温度 |
| `timeout_sec` | integer | DEFAULT `120` | 默认超时（秒） |
| `status` | varchar(32) | NOT NULL, DEFAULT `active`, INDEX | 状态 |
| `is_default` | boolean | DEFAULT `false`, INDEX | 是否为组织默认模型 |
| `is_system` | boolean | DEFAULT `false` | 是否为系统内置配置 |
| `config` | jsonb | | 扩展配置 |

**唯一索引**:
1. `idx_llm_model_org_code`: `(org_id, code)` — 同一组织内编码唯一
2. `idx_llm_model_org_default`: `(org_id)` 带条件 `WHERE is_default = true AND deleted_at IS NULL` — 每个组织最多一个默认模型

---

### 2.9 技能

#### leros_skill — 技能

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | uint | PK | |
| `code` | varchar(255) | UNIQUE, NOT NULL | 技能唯一标识 |
| `org_id` | integer | NOT NULL, INDEX | 所属组织 ID |
| `owner_id` | integer | NOT NULL, INDEX | 拥有者 ID |
| `name` | varchar(255) | NOT NULL | 技能名称 |
| `description` | text | | 技能描述 |
| `icon` | varchar(500) | | 图标 URL |
| `version` | varchar(50) | NOT NULL | 版本号 |
| `category` | varchar(100) | NOT NULL, INDEX | 类别（integration / tool / workflow / ai） |
| `skill_type` | varchar(50) | NOT NULL | 类型（local / remote） |
| `author` | varchar(255) | | 作者 |
| `input_schema` | jsonb | | 输入参数 Schema |
| `output_schema` | jsonb | | 输出参数 Schema |
| `permissions` | jsonb | | 所需权限 |
| `config` | jsonb | | 技能配置 |
| `status` | varchar(50) | NOT NULL, DEFAULT `active` | 状态（active / inactive / deprecated） |
| `is_system` | boolean | DEFAULT `false` | 是否为系统内置技能 |

---

#### leros_skill_registry — 技能注册

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | uint | PK | |
| `skill_id` | integer | UNIQUE, NOT NULL | 关联技能 ID |
| `service_endpoint` | varchar(500) | | 技能服务地址 |
| `execution_path` | varchar(500) | | 技能执行路径 |
| `token` | varchar(255) | | 认证令牌 |
| `status` | varchar(50) | NOT NULL, DEFAULT `registered` | 注册状态（registered / unregistered / unhealthy） |
| `last_heartbeat` | timestamp | | 最后心跳时间 |
| `is_healthy` | boolean | DEFAULT `true` | 健康状态 |

---

#### leros_skill_execution_log — 技能执行日志

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | uint | PK | |
| `skill_id` | integer | NOT NULL, INDEX | 关联技能 ID |
| `session_id` | varchar(255) | INDEX | 执行会话 ID |
| `uin` | varchar(255) | INDEX | 用户 UIN |
| `assistant_id` | varchar(255) | INDEX | 数字助手 ID |
| `input` | jsonb | | 输入参数 |
| `output` | jsonb | | 输出结果 |
| `success` | boolean | | 是否成功 |
| `error_msg` | text | | 错误信息 |
| `duration` | bigint | | 执行耗时（毫秒） |

---

### 2.10 事件

#### leros_event — 外部事件

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | uint | PK | |
| `created_at` | timestamp | | |
| `updated_at` | timestamp | | |
| `deleted_at` | timestamp | nullable | |
| `message_id` | varchar(255) | | 消息唯一标识 |
| `trace_id` | varchar(255) | | 分布式追踪 ID |
| `source` | varchar(255) | | 事件来源（github / gitlab / webhook） |
| `type` | varchar(255) | | 事件类型 |
| `action` | varchar(255) | | 事件动作（opened / closed / updated / commented） |
| `actor` | varchar(255) | | 事件触发者 |
| `target` | varchar(255) | | 事件目标 |
| `payload` | jsonb | | 事件负载 |
| `timestamp` | bigint | | 事件时间戳 |

---

## 3. 关系总览

```
┌──────────────────────┐
│   leros_organization │──┐
└──────────────────────┘  │
                          │ 1:N
┌──────────────────────┐  │
│   leros_project      │──┤
│   leros_task         │──┤
│   leros_session      │──┤
│   leros_digital_     │──┤
│     assistant        │  │
│   leros_skill        │  │
│   leros_artifact     │  │
│   leros_llm_model    │  │
└──────────────────────┘  │
                          │
┌──────────────────────┐  │
│   leros_user         │  │
│   leros_user_org     │──┘
└──────────────────────┘

  leros_project
      │ 1:N
      ├── leros_task
      │       └── leros_artifact (via task_id)
      │
      └── leros_project_member
              │ member_type: user | assistant

  leros_session
      │ 1:N
      └── leros_session_message

  leros_skill
      │ 1:1
      └── leros_skill_registry
      │ 1:N
      └── leros_skill_execution_log
```

---

## 4. 索引汇总

| 表 | 索引 | 字段 | 类型 | 说明 |
|----|------|------|------|------|
| organization | `idx_org_code` | `code` | 唯一 | |
| user | `idx_user_public_id` | `public_id` | 唯一 | |
| user | `idx_user_github_id` | `github_id` | 唯一 | |
| user | `idx_user_github_login` | `github_login` | 唯一 | |
| user_org | `idx_user_org_uin` | `uin` | 唯一 | |
| auth_refresh_token | `idx_auth_token_hash` | `token_hash` | 唯一 | |
| auth_refresh_token | `idx_auth_user_id` | `user_id` | 普通 | |
| auth_login_attempt | `idx_auth_identifier` | `identifier` | 唯一 | |
| digital_assistant | `idx_da_code` | `code` | 普通 | |
| digital_assistant | `idx_da_org_id` | `org_id` | 普通 | |
| project | `idx_project_public_id` | `public_id` | 唯一 | |
| project | `idx_project_org_id` | `org_id` | 普通 | |
| task | `idx_task_public_id` | `public_id` | 唯一 | |
| task | `idx_task_org_id` | `org_id` | 普通 | |
| task | `idx_task_project_id` | `project_id` | 普通 | |
| task | `idx_task_status` | `status` | 普通 | |
| session | `idx_session_public_id` | `public_id` | 唯一 | |
| session | `uni_session_project_task` | `(task_id, project_id)` | 联合唯一 | 允许 NULL |
| session_message | `idx_msg_session_id` | `session_id` | 普通 | |
| session_message | `idx_msg_sequence` | `sequence` | 普通 | |
| artifact | `idx_artifact_public_id` | `public_id` | 唯一 | |
| artifact | `idx_artifact_org_id` | `org_id` | 普通 | |
| artifact | `idx_artifact_task_id` | `task_id` | 普通 | |
| artifact | `idx_artifact_storage_key` | `storage_key` | 普通 | |
| artifact | `idx_artifact_sha256` | `sha256` | 普通 | |
| llm_model | `idx_llm_model_org_code` | `(org_id, code)` | 联合唯一 | |
| llm_model | `idx_llm_model_org_default` | `(org_id)` | 部分唯一 | WHERE is_default AND deleted IS NULL |
| skill | `idx_skill_code` | `code` | 唯一 | |
| skill_registry | `idx_skill_registry_skill_id` | `skill_id` | 唯一 | |

---

## 5. 查询模式参考

### 5.1 获取项目的会话及消息

```sql
SELECT s.*, sm.*
FROM leros_session s
LEFT JOIN leros_session_message sm ON sm.session_id = s.id
WHERE s.project_id = ? AND s.deleted_at IS NULL
ORDER BY sm.sequence ASC;
```

### 5.2 获取任务链

```sql
SELECT t.*, a.*
FROM leros_task t
LEFT JOIN leros_artifact a ON a.task_id = t.id
WHERE t.project_id = ? AND t.deleted_at IS NULL
ORDER BY t.created_at DESC;
```

### 5.3 获取用户默认组织

```sql
SELECT o.*
FROM leros_organization o
JOIN leros_user_org uo ON uo.org_id = o.id
WHERE uo.user_id = ? AND uo.is_default = true AND o.deleted_at IS NULL;
```
---

## 6. 补充表（基于最新代码）

### 6.1 leros_builtin_skill_marketplace_item — 内置技能市场

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `skill_id` | varchar(100) | NOT NULL | 内置技能ID |
| `name` | varchar(255) | NOT NULL | 名称 |
| `description` | text | | 描述 |
| `version` | varchar(50) | NOT NULL | 版本 |
| `author` | varchar(255) | DEFAULT `Leros` | 作者 |
| `category` | varchar(100) | NOT NULL, INDEX | 分类 |
| `tags` | jsonb | | 标签列表 |
| `local_path` | varchar(1000) | NOT NULL | 本地文件路径 |
| `package_sha256` | char(64) | | 包 SHA256 校验 |
| `verified` | boolean | DEFAULT `true` | 是否经过验证 |
| `status` | varchar(50) | DEFAULT `active`, INDEX | 状态 |
| `published_at` | timestamp | | 发布时间 |
| `installs` | bigint | DEFAULT `0` | 安装计数 |

**唯一索引**: `idx_builtin_skill_id_version`: `(skill_id, version)` — 同一技能每个版本唯一

---

### 6.2 leros_file_upload — 文件上传

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `public_id` | varchar(255) | UNIQUE, NOT NULL | 公开ID |
| `org_id` | integer | NOT NULL, INDEX | 组织ID |
| `owner_id` | integer | NOT NULL, INDEX | 拥有者ID |
| `filename` | varchar(500) | | 文件名 |
| `original_name` | varchar(500) | | 原始文件名 |
| `mime_type` | varchar(100) | | MIME类型 |
| `file_size` | bigint | | 文件大小（字节） |
| `storage_path` | varchar(500) | NOT NULL | 存储路径 |
| `sha256` | varchar(64) | INDEX | 文件 SHA256 |
| `purpose` | varchar(50) | INDEX | 用途（attachment / artifact / avatar） |
| `status` | varchar(50) | DEFAULT `active`, INDEX | 状态 |
| `metadata` | jsonb | | 元数据 |

---

## 7. 完整关系总览

```
  组织 (org_id 维度隔离)
  leros_organization
       │
       │ 1:N ─────────────────────────────────────────────────
       │                                                    │
  leros_user_org (user ↔ org 多对多)              leros_digital_assistant
       │                                                    │
       │ N:1                                                 │
  leros_user                                              │
                                                        │
  项目体系                                               │
  ───────                                                │
  leros_project ──── 1:N ──── leros_task ──── 1:N ──── leros_artifact
       │                        │                          │ (via task_id)
       │ 1:N                    │ session_id               │ (via project_id)
       │                        │                          │ (via session_id)
  leros_project_member          │                          │ (via message_id)
       │                        │                          │
       │                        ▼                          │
       ▼              leros_session ──── 1:N ────┐         │
  member_type:                           leros_session_message
  user | assistant                              │
                                                │ chunks (jsonb)
                                                │ artifacts (jsonb)
                                                │ usage (jsonb)

  技能体系
  ───────
  leros_skill ──── 1:1 ──── leros_skill_registry
       │
       └── 1:N ──── leros_skill_execution_log
                         │ skill_id
                         │ session_id / uin / assistant_id

  内置技能市场
  leros_builtin_skill_marketplace_item

  LLM 模型配置
  leros_llm_model
       │ org_id + code 联合唯一
       │ org_id 部分唯一 (is_default = true)

  认证体系
  ───────
  leros_user
       │ 1:N
       ├── leros_auth_refresh_token (token_hash 唯一)
       └── leros_auth_login_attempt (identifier 唯一)

  事件存储
  leros_event (外部事件持久化)

  文件上传
  leros_file_upload (文件存储记录)
```

---

## 8. 命名约定

| 前缀 | 用途 | 示例 |
|---|---|---|
| `prj_` | 项目公开ID | `prj_3x8F2kL9` |
| `task_` | 任务公开ID | `task_7mN1pQ4R` |
| `sess_` | 会话公开ID | `sess_5vB2wX8Y` |
| `req_` | 请求ID | `req_9cD3eF6G` |

ID 生成使用 Snowflake 算法（`ygpkg/yg-go/encryptor/snowflake`），Base58 编码。

---

## 9. JSONB 字段详解

### 9.1 session_message.chunks — 流式消息片段

```json
[
  {
    "seq": 1,
    "last_seq": 10,
    "type": "text",
    "timestamp": 1718000000000,
    "payload": { /* json.RawMessage */ }
  }
]
```

### 9.2 session_message.artifacts — 产物引用

```json
[
  {
    "artifact_id": 1,
    "title": "report.pdf",
    "filename": "report.pdf",
    "mime_type": "application/pdf",
    "artifact_type": "file"
  }
]
```

### 9.3 session_message.usage — Token 用量

```json
{
  "input_tokens": 1500,
  "output_tokens": 800,
  "total_tokens": 2300
}
```

### 9.4 session_message.attachments — 文件附件

```json
[
  {
    "file_upload_id": "upload_xxx",
    "name": "image.png",
    "mime_type": "image/png",
    "size": 102400
  }
]
```

---

## 10. 索引汇总

| 表 | 索引名 | 字段 | 类型 | 说明 |
|----|--------|------|------|------|
| organization | `idx_org_code` | `code` | 唯一 | |
| user | `idx_user_public_id` | `public_id` | 唯一 | |
| user | `idx_user_github_id` | `github_id` | 唯一 | |
| user | `idx_user_github_login` | `github_login` | 唯一 | |
| user_org | `idx_user_org_uin` | `uin` | 唯一 | |
| auth_refresh_token | `idx_auth_token_hash` | `token_hash` | 唯一 | |
| auth_refresh_token | `idx_auth_user_id` | `user_id` | 普通 | |
| auth_login_attempt | `idx_auth_identifier` | `identifier` | 唯一 | |
| digital_assistant | `idx_da_code` | `code` | 普通 | |
| digital_assistant | `idx_da_org_id` | `org_id` | 普通 | |
| project | `idx_project_public_id` | `public_id` | 唯一 | |
| project | `idx_project_org_id` | `org_id` | 普通 | |
| task | `idx_task_public_id` | `public_id` | 唯一 | |
| task | `idx_task_org_id` | `org_id` | 普通 | |
| task | `idx_task_project_id` | `project_id` | 普通 | |
| task | `idx_task_status` | `status` | 普通 | |
| session | `idx_session_public_id` | `public_id` | 唯一 | |
| session | `uni_session_project_task` | `(task_id, project_id)` | 联合唯一 | 允许NULL |
| session_message | `idx_msg_session_id` | `session_id` | 普通 | |
| session_message | `idx_msg_sequence` | `sequence` | 普通 | |
| artifact | `idx_artifact_public_id` | `public_id` | 唯一 | |
| artifact | `idx_artifact_org_id` | `org_id` | 普通 | |
| artifact | `idx_artifact_task_id` | `task_id` | 普通 | |
| artifact | `idx_artifact_storage_key` | `storage_key` | 普通 | |
| artifact | `idx_artifact_sha256` | `sha256` | 普通 | |
| llm_model | `idx_llm_model_org_code` | `(org_id, code)` | 联合唯一 | |
| llm_model | `idx_llm_model_org_default` | `(org_id)` | 部分唯一 | `WHERE is_default=true AND deleted_at IS NULL` |
| skill | `idx_skill_code` | `code` | 唯一 | |
| skill_registry | `idx_skill_registry_skill_id` | `skill_id` | 唯一 | |
| file_upload | `idx_file_upload_public_id` | `public_id` | 唯一 | |
| builtin_skill_marketplace | `idx_builtin_skill_id_version` | `(skill_id, version)` | 联合唯一 | |

---

## 11. 查询模式参考

### 11.1 获取项目的会话及消息

```sql
SELECT s.*, sm.*
FROM leros_session s
LEFT JOIN leros_session_message sm ON sm.session_id = s.id
WHERE s.project_id = ?
  AND s.deleted_at IS NULL
ORDER BY sm.sequence ASC;
```

### 11.2 获取任务链（含产物）

```sql
SELECT t.*, a.*
FROM leros_task t
LEFT JOIN leros_artifact a ON a.task_id = t.id
WHERE t.project_id = ?
  AND t.deleted_at IS NULL
ORDER BY t.created_at DESC;
```

### 11.3 获取用户默认组织

```sql
SELECT o.*
FROM leros_organization o
JOIN leros_user_org uo ON uo.org_id = o.id
WHERE uo.user_id = ?
  AND uo.is_default = true
  AND o.deleted_at IS NULL;
```

### 11.4 获取 Task 关联的 Session 完整消息

```sql
SELECT s.*, sm.*
FROM leros_session s
JOIN leros_session_message sm ON sm.session_id = s.id
WHERE s.task_id = ?
  AND s.type = 'task'
  AND s.deleted_at IS NULL
ORDER BY sm.sequence ASC;
```

### 11.5 获取组织默认 LLM 模型

```sql
SELECT * FROM leros_llm_model
WHERE org_id = ?
  AND is_default = true
  AND deleted_at IS NULL;
```

### 11.6 查询用户的活跃 Worker（数字员工）

```sql
SELECT da.*
FROM leros_digital_assistant da
JOIN leros_session s ON s.allocated_assistant_id = da.id
WHERE s.uin = ?
  AND s.status = 'active'
  AND da.deleted_at IS NULL
GROUP BY da.id;
```

---

## 12. 设计要点

### 12.1 多租户隔离

通过 `org_id` 字段实现租户隔离。几乎所有业务表都包含 `org_id`，并在查询时作为过滤条件。

### 12.2 软删除与唯一索引

使用 GORM 的 `deleted_at` 软删除机制。唯一索引需要特别注意：GORM 在创建唯一索引时会自动添加 `WHERE deleted_at IS NULL` 条件（以支持软删除场景下的唯一约束）。

特例：`uni_session_project_task` 索引涉及可为 NULL 的字段，GORM 的 MySQL 驱动对 NULL 联合唯一索引有特殊处理。

### 12.3 JSONB 字段

大量使用 JSONB 存储半结构化数据：metadata、chunks（流式片段）、artifacts（产物引用）、usage（Token 用量）、attachments（文件附件）、input_schema/output_schema（技能 Schema）、permissions（权限）、config（配置）等。这反映了系统对灵活性的需求——核心业务结构固定，扩展信息使用 JSONB。

### 12.4 公开 ID 设计

内部使用自增 `uint` 主键，对外暴露使用 Snowflake 生成的公共 ID（如 `sess_xxx`、`prj_xxx`、`task_xxx`）。这样可以：
- 避免暴露自增 ID 带来的安全风险（遍历猜解）
- 提供有语义的前缀，便于识别对象类型
- 支持分布式生成，无需中心化 ID 服务

### 12.5 Session 的多作用域

Session 有三种作用域：
- **Project Session**：项目级别的会话，`type=project`，绑定 `project_id`
- **Task Session**：任务级别的会话，`type=task`，绑定 `project_id` + `task_id`
- **Standalone Session**：独立会话，仅绑定用户

`(task_id, project_id)` 的联合唯一索引确保一个任务只能有一个关联的 Session。
