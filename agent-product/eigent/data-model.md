# Eigent 数据模型 / 存储模块深度解读

> 本篇解读对象：Eigent server（`server/`）控制面的持久化层。仓库根 `/Users/morehao/Documents/study/ai/agent-product/eigent`，源码路径均以此为根。
>
> 本文档回答的根问题：这个 Multi-Agent Workforce 的**状态到底存在哪、表怎么设计、怎么演进、读写怎么走**——即"控制面 server 管所有持久化状态"这句话在存储层面的具体落地。

---

## 设计哲学与本模块的定位

三端架构里，backend（CAMEL runtime）是行动面，被刻意设计成**对每个任务尽量无状态**；桌面端是交互面。所有需要跨任务存活、需要被多端共享、需要审计/回放的状态——用户、工作区、会话与逐步执行、MCP 连接、定时自动化、记忆、账单——都收敛到 server 控制面的关系数据库。数据模型就是这张"状态总账"的账簿：**它不是某个功能的附属，而是所有 domain（workforce/memory/mcp/trigger/project）共用的地基**。

数据模型文档最容易退化成"字段字典"，本篇按设计五要素展开：实体关系 → 存储选型 → 核心表设计 → Schema 演进 → 读写路径，最后回扣动机。

---

## 实体关系总览

`server/app/model/` 下共 33 个 `table=True` 实体。核心业务的 ER 关系如下（省略了账单/admin 等支撑表，见文末覆盖率表）：

```mermaid
erDiagram
    User ||--o{ Space : owns
    Space ||--o{ Project : contains
    Project ||--o{ ChatHistory : produces_task
    Project ||--o{ Trigger : automates
    Project ||--o{ ProjectMemory : scoped_knowledge
    Space ||--o{ SpaceMemory : shared_knowledge
    Space ||--o{ SpaceFileIndex : file_cache
    Project ||--o{ SpaceFileIndexOverlay : pending_diff
    Trigger ||--o{ TriggerExecution : runs
    Category ||--o{ Mcp : categorizes
    Mcp ||--o{ McpEnv : env_template
    Mcp ||--o{ McpUser : per_user_install
    User ||--o{ McpUser : installs
    ChatHistory ||--o{ ChatStep : transcript
    User ||--o{ RemoteControlSession : owns
    RemoteControlSession ||--o{ RemoteControlCommand : emits
    User ||--o{ Provider : model_provider
    User ||--o{ Config : toolkit_credentials
    User ||--o{ UserCreditsRecord : credit_ledger

    User {
        int id PK
        string email UK
        string username UK
        int credits
    }
    Space {
        string id PK
        string user_id FK
        string source_type
        int schema_version
        json metadata
    }
    Project {
        string id PK
        string user_id
        string space_id FK
        string mode
        string status
        string workdir_mode
    }
    ChatHistory {
        int id PK
        string task_id UK
        string project_id
        string space_id
        string run_id
        json installed_mcp
        json skip_reason
    }
    ChatStep {
        int id PK
        string task_id
        string run_id
        json data
    }
    Trigger {
        int id PK
        string user_id
        string project_id
        string space_id
        json config
    }
    TriggerExecution {
        int id PK
        int trigger_id FK
        string execution_id UK
        json input_data
        json output_data
    }
```

关系要点：

- **User 是全局租户根**，下拉 Space；Space 是"共享源容器"（一个 Space 被多个 Project 引用）；Project 是"会话式执行容器"。这是 Eigent 从单用户时代演进出的 workspace 分层（见演进章节）。
- **ChatHistory 与 ChatStep 之间不是数据库外键**，而是通过 `task_id`/`run_id` 两个业务键松耦合——这是刻意的去规范化，因为 step 由 backend 高频写入、不便 join，用等值查询而非关系拉取。
- **每个新用户的 [User, Space, Project] 链条并非预建**，而是在第一次写 chat_history 时按需 `ensure` 出来（见读写路径）。

---

## 存储选型与理由

### 用的是关系型 ORM：SQLModel(+SQLAlchemy) + 默认 PostgreSQL

`database.py` 只做一件事——按环境变量建 engine：

```python
engine = create_engine(
    env_or_fail("database_url"),
    echo=True if env("debug") == "on" else False,
    pool_size=36,
)
```

`database_url` 来自环境，docker-compose 注入的是 PostgreSQL（`postgresql://postgres:123456@postgres:5432/eigent`），`alembic.ini` 内默认也是同样的 PostgreSQL URL。所以**默认/生产形态是 PostgreSQL，SQLite 只是"换 URL 即可切换"的兼容路径**，并非 SQLite 优先。

选关系型的理由，对照不选方案的代价：

| 不选的方案 | 代价（为什么不要） |
|---|---|
| 纯文档库（MongoDB/NoSQL） | 关系型天然承接 `User→Space→Project` 这套强分层 + 唯一约束（task_id UK、space_id+key UK）+ 事务化回填。文档库做这些要么靠应用层补偿，要么靠稀疏索引，一致性更累。 |
| 纯键值（Redis/NoKV-only） | 需要 `case when`、`count`、聚合分组（grouped history），键值模型难以表达列式过滤与统计。 |
| 文件/纯 JSON 落盘 | 多端并发读写、tokens/spend 聚合统计、软删除与审计都不好做。 |

但 Eigent 并没"全押关系列"，而是**关系列承载身份/关系/可索引字段，动态载荷一律塞 JSON blob**（`metadata`、`installed_mcp`、`data`、`env`、`config`、`payload`、`root_fingerprint` 全都有 JSON 列）。这实质是**"关系骨架 + 文档载荷"的混合取向**：把 schema 中最不稳定、最易演进的部分隔离进 JSON，避免为它们频繁开迁移。这是整个存储设计里最一致的风格，见演进章节进一步展开。

### 为什么 SQLModel 而非纯 SQLAlchemy

`AbstractModel` 直接继承 `SQLModel`（[server/app/model/abstract/model.py:43]()），实体以 `table=True` 声明。选型收益是**单类双态**：一个 Pydantic+SQL 融合模型同时当数据库实体和 API 入参/出参用——每个模型文件里 `XxxIn/XxxOut`（Pydantic）都写在 `table=True` 实体旁边，`ChatHistory` 既是 ORM 又是 `ChatHistoryOut`/`ChatHistoryIn` 的字段来源。

这是对 FastAPI 控制面的天然亲和：**models 即 schema 即 validation**，省掉"ORM→DTO→Pydantic"的三层样板。代价是 SQLModel 本身就是对 SQLAlchemy 的一个"pragmatic 子集"，复杂查询仍要回退 `session.exec(select(...))`——`AbstractModel` 提供的 `by/exists/count/delete_by` 正是把这些读路径封装好，让 domain 层不必直接碰 SQLAlchemy（[server/app/model/abstract/model.py:49-122]()）。

### 一致性 / 本地优先的取舍

Eigent 明确选的是**控制面全状态集中、backend 无状态**。集中意味着：

- 迁移、备份、审计只在 server 一处做——单机控制面 DB 是本地优先的代价，换来的却是 schema 演进自由度（见演进章节，space 层上线是一次大迁移）。
- 代价是**每一笔 step 写入都要过 HTTP 落到 DB**（高写入点），所以 ChatStep 不设外键、只用 `task_id` 索引等值查询，淡化关系约束以换吞吐。

---

## 核心表设计

> 每个模型文件都自带一套 `XxxIn/XxxOut`（Pydantic），以下字段表聚焦 `table=True` 的 ORM 实体；`源` 列给到类定义所在文件行。

### AbstractModel 与 DefaultTimes：所有表的公共基因

`AbstractModel(SQLModel)`（[server/app/model/abstract/model.py:43]()）提供三样东西：**自动表名**（`snake_case(cls.__name__)`，`:45`）、**一组查询封装**（`by/exists/count/exists_must/delete_by`，`:49-122`）、**统一的保存/删除姿势**（`save` `:123` / `delete` `:148`）。

`DefaultTimes`（`:174`）是软删除+时间戳的混入，被几乎所有业务实体继承：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `deleted_at` | `datetime?` | 默认 `None` | 软删除标记；非空即代表已删除 |
| `created_at` | `TIMESTAMP?` | `server_default=CURRENT_TIMESTAMP` | 建行时间，DB 侧默认兜底（兼容 MySQL） |
| `updated_at` | `TIMESTAMP?` | `server_default` + `onupdate=func.now()` | 行变更自动刷新，可感知最后一次修改 |

关键机制在 `delete()`：**对继承 DefaultTimes 的实体做软删除（置 `deleted_at`），否则物理删除**（[server/app/model/abstract/model.py:157-162]()）。软删除的取舍是：历史 step/会话可保留审计与回放，代价是每个查询都要带 `no_delete()` 过滤（`:191-193` 提供 `deleted_at IS NULL` 谓词）。这是 Eigent"过程可回放/教学用料优先"的体现——删除只是逻辑隐藏，不是物理抹除。

### User：全局租户根

`User`（[user/user.py:32]()）是唯一被多个 domain 外键引用的实体，作为租户根。

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | int | PK autoincrement | 部门级 int 主键 |
| `email` | EmailStr | UK, not null | 登录唯一标识 |
| `username` / `stack_id` | str? | UK | 备用身份，可空 |
| `password` | str? | max256 | 可空（支持非密码登录） |
| `credits` | int | `server_default=0` | 可消费的词元余额，**冗余聚合在 User 行** |
| `last_daily/monthly_credit_date` | date? | — | 每日/每月赠送计提游标，防重复发币 |
| `inviter_user_id` | int? | FK→user.id | 邀请链自引用 |
| `status` | `ChoiceType(Status,SmallInteger)` | — | Normal=1/Block=-1，整型压缩 |

设计点：
- **`credits` 冗余到 User 行**而不是只存流水——读余额零 join；真实账本在 `UserCreditsRecord`（见 pay 章节），`User.credits` 是它的投影，靠领域服务保证最终一致。
- status 用 `ChoiceType(Status, SmallInteger())` 存**小整数而非字符串**——这是全仓一致的风格，用整数压缩存储与索引，语义通过 IntEnum 映射。

### Space / Project：workspace 分层

`Space`（[space/space.py:37]()）是"被多个 project 共享的顶层源容器"。`Project`（[project/project.py:42]()）是"挂在 Space 下的会话式执行容器"，本质是**一条聊天任务链 + 一个 workdir 的绑定**。

**Space**：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | **str** | PK | **演进产物：Space 层新建后用字符串主键** |
| `user_id` | str(128) | not null, index | 拥有者 |
| `name` / `description` | str | not null | 展示名 |
| `source_type` | str(50) | not null, index, CHECK(`blank/folder/legacy`) | blank=只 artifact；folder=绑定真实目录；legacy=迁移来的历史空间 |
| `root_path` | str(2048)? | — | folder 空间的源目录 |
| `root_fingerprint` | JSON? | — | 目录身份指纹（device/inode），云/多机部署时服务器无法 stat 本地路径的兜底 |
| `status` | str(50) | not null, index, CHECK(`active/disconnected/archived`) | disconnected=源目录掉线 |
| `schema_version` | int | `server_default=1` | **显式载荷版本号**，读取时可据此解释 metadata（演进安全关键） |
| `metadata` | JSON? | 列名 `metadata` | 扩展载荷 |

**Project**：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | **str** | PK | 同 Space，字符串 id |
| `user_id` | str(128) | not null, index, UK(`user_id,id`) | 复合 UK：**同一 id 可被不同用户复用**，隔离租户 |
| `space_id` | str | FK→space.id, index | 归属空间 |
| `mode` | str(50)? | index, CHECK(`single-agent/workforce`) | 执行模式，首次学习后不覆盖 |
| `status` | str(50) | not null, index, CHECK(`active/archived`) | 生命周期 |
| `workdir_mode` | str(50)? | index, CHECK(`worktree/copy/direct-write/artifact-only`) | 工作目录处理策略，是 Eigent 的 Apply 能力根 |
| `metadata` | JSON? | 列名 `metadata` | 扩展载荷 |

设计点：space/project 都是**字符串主键**——`Space.id`/`Project.id` 没有 `max_length` 元数据约束，语义上允许由客户端生成/复用（如 `legacy_42`、`user:project` 复合键）。这直接服务两条动机：① legacy 数据回填需要"可预测 id"（见演进）；② HistoryGroup 需要按字符串 project_id 分组，字符串 id 免去类型转换。复合 UK(`user_id, id`) 意味着**不同用户可有相同 project_id，靠 user_id 隔离**——这是多租户落在 schema 上的直接体现。

### ChatHistory / ChatStep：执行台账

**ChatHistory**（[chat/chat_history.py:33]()）是"一次跑的任务头部"：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | int | PK | |
| `user_id` | int | index | 归属用户 |
| `task_id` | str | **index, UK** | **任务唯一键**，去重/幂等靠它 |
| `project_id` | str? | index, 空时回退 task_id | 新迁移字段，链回 project |
| `space_id` / `run_id` | str? | index | space 层迁移新增，run 生命周期 |
| `question` / `language` / `model_platform` / `model_type` | str | not null | 请求上下文快照 |
| `installed_mcp` | JSON | 默认 `{}` | 该任务装的 MCP 集合快照 |
| `tokens` / `spend` | int/float | `server_default=0` | 用量成本，供统计/账单 |
| `status` | `ChoiceType(ChatStatus,SmallInteger)` | — | ongoing=1/done=2 |
| `skip_reason` | JSON? | space 迁移新增 | 结构化"为何未执行"（space 掉线/冲突等） |

**ChatStep**（[chat/chat_step.py:25]()）是"逐步执行节点"，后端高频写入：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | int | PK | |
| `task_id` | str | index | 与历史头关联的业务键 |
| `run_id` | str? | index | 一次 run 内分段，space 迁移新增 |
| `step` | str | — | 步骤名/类型 |
| `data` | **JSON** | — | 步骤结构化载荷（模型就把 dict/list 预序列化再入库，pydantic validator 兜底） |
| `timestamp` | float? | — | 墙钟戳，排序用 |

设计点：**ChatStep.data 是纯 JSON**，因为它承载"任意 agent 步骤产物"，schema 根本无法预先列化——这正是上一节"JSON blob 吸收不稳定载荷"原则的极端形态。ChatStep 不设外键、只用 task_id/run_id 索引，因为它是最高频写入点（每步一写），用等值索引避免 join 放大写放大。

### Mcp / McpUser / McpEnv：MCP 工具目录与按需安装

MCP 是三层：**平台目录**（`Mcp`，[mcp/mcp.py:43]()，全局标准工具）→ **环境模板**（`McpEnv`，预置 env 默认值）→ **用户安装**（`McpUser`，把某工具装到某个 user，含自定义 env/命令）。

**Mcp**：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | int | PK | |
| `category_id` | int | FK→category.id | 分类归属 |
| `name` / `key` / `description` | str | — | 展示与程序键 |
| `type` | `ChoiceType(McpType,SmallInteger)` | — | Local=1/Remote=2 |
| `status` | `ChoiceType(Status,SmallInteger)` | — | Online=1/Offline=-1 |
| `install_command` | JSON | — | `{command,args,env}` 安装命令模板（如 uvx 启动 mcp-server-everything-search） |

**McpUser**（[mcp/mcp_user.py:41]()）是平台 MCP 与用户之间的关联表，但被设计成**富关联**：冗余了 `command/args/env/type/status/server_url`，等于给"用户 A 的这个 MCP 实例"存了独立运行参数——这是多对多+n + 每边属性的一体化，免去再开一张参数表。`mcp_id→mcp.id`、`user_id→user.id` 双外键，构成多对多。

设计点：`Mcp.install_command` 用 JSON 存结构化安装指令而不是拆三列——同一理由，指令形状演进自由。

### Trigger / TriggerExecution：定时自动化

`Trigger`（[trigger/trigger.py:24]()）把"何时/何条件触发一条任务"持久化；`TriggerExecution`（[server/app/model/trigger/trigger_execution.py:25]()）记每次触发运行。

**Trigger**（自动化定义）：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | int | PK | |
| `user_id` / `project_id` / `space_id` | str | index | 归属链（space 迁移新增 space_id） |
| `trigger_type` | `ChoiceType(TriggerType,String)` | **用 String** | schedule / webhook / slack_trigger，枚举名存字符串 |
| `status` | `ChoiceType(TriggerStatus,String)` | — | active/inactive… |
| `webhook_url` / `webhook_method` | str | — | webhook 类型专属，自动生成 URL + method |
| `custom_cron_expression` | str? | — | schedule 类型 cron |
| `config` | JSON? | — | **触发类型专属配置**（由 app_configs 按类型校验） |
| `max_executions_per_hour/day` | int? | — | 限流 |
| `is_single_execution` | bool | — | 只跑一次 |
| `next_run_at` | datetime? | index | 调度器扫这个索引抓"到点了"的触发 |
| `consecutive_failures` / `auto_disabled_at` | int/datetime? | — | 失败熔断，自动禁用 |

**TriggerExecution**（触发运行日志）：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | int | PK | |
| `trigger_id` | int | FK→trigger.id, index | 归属触发 |
| `execution_id` | str | **index, UK** | 触发实例唯一键，幂等落库 |
| `execution_type` / `status` | ChoiceType(String) | — | scheduled/webhook；pending… |
| `started_at`/`completed_at`/`duration_seconds` | datetime/float | — | 时长 |
| `input_data` / `output_data` | JSON | — | 触发入参/产出 |
| `attempts` / `max_retries` | int | 默认 1/3 | 重试状态 |
| `tokens_used` | int? | — | 成本 |

Trigger 的 `config` JSON 由 `trigger/app_configs/` 按 `trigger_type` 选 Pydantic 校验类——[server/app/model/trigger/app_configs/config_registry.py:35-39]() 把 schedule/webhook/slack 映射到对应 config 类，`base_config.py` 提供公共校验（regex 过滤、失败上限、凭据激活检查）。这是 **JSON blob + 强类型验证器**的组合：DB 层不锁 schema，验证层锁语义。

> 一致性注记：Trigger 系列的枚举用 `ChoiceType(..., String)` 存字符串（schedule、webhook），而 Mcp/User/ChatHistory 用 SmallInteger——**全仓枚举序列化并不统一**，是演进中"新增模块自带风格"留下的痕迹；字符串可读但比整数占位大。

### Memory：Space / Project 两级知识

`memory/memory.py` 提供两个对称的 KV 表：`SpaceMemory`（空间级共享知识）与 `ProjectMemory`（会话级知识）。两者结构一致：

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | int | PK | |
| `user_id` | str(128) | not null, index | 归属 |
| `space_id` / `project_id` | str | FK, index | 作用域 |
| `key` | str(255) | not null, index, **UK(scope_id, key)** | 知识键，scope 内唯一 |
| `value` | JSON? | — | 知识值 |
| `metadata` | JSON? | 列名 `metadata` | 扩展 |

设计点：UK 落在 `(scope_id, key)`——同一个 key 在多个空间/项目间天然隔离，这是"作用域隔离"的一等公民设计；value 用 JSON 承载任意类型知识。

### file_index / overlay / Apply：目录懒缓存与制品 pending diff

`space/file_index.py` 两个表服务"Source 空间 → 制品"的**懒缓存**：

- `SpaceFileIndex`（`:30`）：Space 源文件的缓存行（path/hash/size/mode/modified_at），`UK(space_id, path)` 防重复，含 `row_version` 乐观锁字段，`indexed_at` 记录扫描时间。类 docstring 明说："Lazy cache of Space source files. Apply must still verify live disk."——**缓存不代替代磁盘真源校验**，一致性的最终裁决仍交给实时 stat。
- `SpaceFileIndexOverlay`（`:49`）：run 级 pending diff 表，`UK(space_id, project_id, run_id, path)`，记录 agent 对源文件的 `added/modified/deleted` 三类改动，`base_hash`/`base_snapshot_id` 支撑 diff 基线。这是 Eigent 的 Apply 能力（把 agent 制品写回源空间）在存储层的根基——agent 改动先落 overlay，用户确认后才 apply。

对应的 `space/apply.py` 全是一组 Pydantic 请求/响应模型（ApplyResolutionIn/Conflict/Result），它们消费上面两张表，体现"模型层只管结构，行为在 service"的分层。

### 支撑表（简述）

- `config/config.py`：统一定义"工具/触发器的凭据存放处"——`Config(user_id, config_name, config_value, config_group)`，`UK(user_id, config_name)`；`ConfigInfo` 用静态 dict 把 **ConfigGroup → 需要的 env_vars/toolkit/trigger** 声明式映射（`:51-179`），供 trigger 激活时校验凭据是否齐备（base_config.py:154-229 查这张表）。
- `user/` 下 `Admin`/`AdminRole`/`Role`（后台管理员与权限，AdminRole 是 admin 与 role 的多对多中间表）、`Key`（内置 LLM 网关 key）、`UserPrivacy`（1:1 隐私设置，`UK(user_id)`）、`UserCreditsRecord`（credit 流水账本，`user_credits.py` 只定义 Channel 枚举、表是 [server/app/model/user/user_credits_record.py:53]()）、`UserStat`（产品页/支付统计聚合）。
- `pay/order.py`：`Order`（订单表，`user_id→user.id`、`plan_id→plan.id`），带 `mark_success/mark_failed/mark_pending` 状态机方法与 `create_addon_order`/`create_plan_order` 工厂。
- `provider/provider.py`：`Provider`（模型供应商，含 `encrypted_config` JSON 加密载荷、`prefer` 首选标记）。
- `remote_control/remote_control.py`：`RemoteControlSession/Link/Command/Event` 四张表，支撑 Brain 模式远程设备接管，`id` 均为 String(64) 主键，`expires_at`/`bridge_status` 反映会话心搏。
- `remote_sub_agent/provider.py`：`RemoteSubAgentProvider`（远程子 agent 供应商，`enabled` 需 api_key/endpoint/agent_name 齐备）。

---

## Schema 演进策略

演进机制分三层：**Alembic 迁移（应用外启动） + JSON blob 吸收变化 + 程序化数据回填/冲突消解**。

### 迁移机制：Alembic，启动脚本而非应用内执行

已通过程序化线索确认（grep `create_all/alembic/migrate` + 读 `alembic.ini`/`env.py`/`versions/`）：`server/alembic.ini` 存在、`server/alembic/versions/` 下有 10 个迁移文件、`server/alembic/env.py` 用 `SQLModel.metadata` + `auto_import` 注册各 model 包。迁移命令在**容器/启动脚本**执行，而非应用启动代码：

```sh
# 等待数据库就绪...
uv run alembic upgrade head
exec uv run uvicorn main:api --host 0.0.0.0 --port 5678
```

（`server/start.sh` 片段；Dockerfile `CMD ["/app/start.sh"]`。）所以**建表入口 = `alembic upgrade head`，运行时代码不做 create_all**——schema 归迁移管，代码只管读写。`alembic/env.py` 通过 `auto_import`（[server/app/core/environment.py:79-91]()` 逐个 import 目录下所有 .py）把所有 model 包（mcp/user/config/chat/provider/remote_sub_agent/remote_control/trigger/space/project/memory）注册进 `SQLModel.metadata`，autogenerate 时能对齐全部实体。

### 演进安全的一个关键取舍：忽略外键约束

`alembic/env.py` 的 `include_object` 对 `foreign_key_constraint` 一律返回 `False`（[server/alembic/env.py:118-122]()）——**autogenerate 不跟踪、迁移也不生成/删外键**。这是刻意的权衡：

- 好处：schema diff 时不被"加了/删了 FK"这类低风险变化干扰。
- 代价：外键约束在演进中**不会自动建/删**，全凭模型的显式 `foreign_key=...` 声明 + 初始迁移生成，之后演进不重管。

这解释了为什么 `ChatStep→ChatHistory` 等热路径根本不建 FK、只用 task_id 索引——**一致性交给应用层，换来演进与写的自由**。

### 最重的一笔：space 层上线的程序化数据回填

`2026_05_25_1200-add_space_layer_foundation.py` 是一次教科书级的"大重构 + 数据迁移"。它不只是 `create_table`，而是把**单用户扁平模型在线折叠进 workspace 分层**，包含：

1. 新建 `space/project/space_memory/project_memory/space_file_index(+overlay)/user_id_canonicalization` 表（`:54-242`）。
2. 给存量 `chat_history/chat_step/chat_snapshot/trigger/trigger_execution` **加列**（space_id/run_id/storage_key/skip_reason，`:244-259`）。
3. **程序化回填**：先给旧 chat_history 塞兜底 `space_id='legacy_'+user_id`（`:265`）、`project_id=task_id`（`:261`）、`run_id=task_id`（`:262`），保证整表形状统一；再建 `user_id_canonicalization` 把各表面 user ids 统一到 TEXT uid（`:274-289`）；再为每个用户造一个 `legacy_` Space（`:291-312`）；最后用 CTE 从 chat_history+trigger 聚合出 project 并回填（`:314-379`）。
4. **冲突消解**：同一 project_id 被多用户占用时，用 `user_id:project_id` 复合串消歧，并在 project.metadata 存 `legacyAlias`，随后把 chat_history/trigger 的 project_id 一并更新对齐（`:342-406`）。

这段证明了 schema 演进的第三条腿：**不止加表加列，还要在迁移内做数据搬移 + 冲突消解**，且整体幂等（`ON CONFLICT ... DO NOTHING`）。这是"迁移安全"的真正实现——不是靠工具自动，而是靠迁移脚本内显式的回填 CTE。

### JSON blob + 提升列的渐进式加字段

演进常态是：**不确定形状的字段先落 JSON，需求稳定后再提升为真列**。

- 每个实体都带 `metadata` JSON 列，新业务字段先放这里；读侧用 `schema_version`（如 `Space.schema_version`）决定如何解释。`root_fingerprint`、`skip_reason` 等都是"先在 JSON、后提升为命名 JSON 列"的实例。
- 已提升的列（chat_history 的 space_id/run_id/project_id）能看到从衍生值 → 真列 + 索引 + 回填的完整路径（迁移 `:244-248`）。

风险与代价：JSON 列无法被 SQL 高效过滤/索引，`grouped history` 的分组只能依赖已提升的 project_id 列，而非 metadata 内嵌值——作者在这条线上做了清醒的取舍。

### 回滚与幂等

每个迁移都有对称 `downgrade()`（space 迁移 `:409` 起逐个 drop_column/drop_table）。回填语句用 `INSERT ... ON CONFLICT DO NOTHING` 保证重跑幂等。

---

## 读写路径

### "创建一次会话历史"的写入链路

后端 agent 完成一段任务 → 把头部信息 POST 到控制面 → server 先 ensure Space/Project，再落 ChatHistory 一行：

```mermaid
sequenceDiagram
    participant B as backend
    participant C as history_controller
    participant S as SpaceService
    participant DB as PostgreSQL
    B->>C: POST /chat/history
    C->>C: 补默认值(project_id, run_id, space_id)
    C->>S: ensure_project(user, project, space, name)
    S->>DB: SELECT space WHERE id=? 
    S->>S: 不存在则 ensure_legacy_space 建行
    S->>DB: SELECT project WHERE id=? 且 user_id=?
    S->>S: 不存在则 begin_nested 建 project
    S-->>C: Project 存在或新建成功
    C->>DB: INSERT chat_history
    C->>DB: COMMIT (IntegrityError 则 409)
    DB-->>C: 返回 chat_history (已 refresh)
    C-->>B: ChatHistoryOut
    ```

关键引用：
- 默认值补齐与 ensure：[server/app/domains/chat/api/history_controller.py:97-114]()
- `ensure_legacy_space` 惰性建 space，带 `begin_nested` 防并发撞车：[server/app/domains/space/service/space_service.py:152-180]()
- `ensure_project` 同样用 `begin_nested` + 捕获 `IntegrityError` 自愈重试（幂等）：[server/app/domains/space/service/space_service.py:466-505]()
- 软删除/`no_delete()` 保证读默认过滤已删行：[server/app/model/abstract/model.py:191-193]()

### "逐步执行节点"的高频写入链路

每执行一步，backend 就把该步抛给 `step_controller` 落一行 ChatStep：

```python
chat_step = ChatStep(
    task_id=step.task_id,
    run_id=step.run_id or step.task_id,
    step=step.step,
    data=step.data,
    timestamp=step.timestamp,
)
db_session.add(chat_step)
db_session.commit()
```

（[server/app/domains/chat/api/step_controller.py:208-217]()。）写入后还异步 `publish_chat_step` 广播给远程控制端（`:219`）——DB 写入与事件广播在同一请求内，广播失败仅告警不阻断落库。这展示了控制面"状态入库主链 + 副作用广播"的解耦。

### 读取回传：grouped history

`GET /chat/histories/grouped` 走 `ChatService.get_grouped_histories`，在库内按 project_id 分组聚合 tokens/count，回传结构化 `ProjectGroup`（[server/app/domains/chat/api/history_controller.py:142-149]() → `chat_history_grouped.py`）。排序对**老数据无 created_at** 的情况回退 `CASE ... desc(id)`（[server/app/domains/chat/api/history_controller.py:130-138]()），保证迁移前数据也稳定排序——这是对"迁移可能落后于新代码"的读侧兜底。

---

## 存储这块与整体设计哲学的全局关联

每类数据连回它服务的业务能力：

- **User(credits) + UserCreditsRecord(账本) + Order(pay)**：服务商业化——credits 是行动面的"能量来源"，pay 让 credits 可售。控制面管账，backend 只消耗。
- **Space/Project + SpaceFileIndex/Overlay(Apply)**：服务 **Workforce 的隔离与制品管理**——每个 project 一个 workdir，overlay 记"agent 对源文件未写入的 pending diff"，回连 `workdir_mode` 四种策略。
- **ChatHistory/ChatStep**：服务**会话状态 + 逐步回放/审计**，也是三端共享的任务台账；软删除保住可回放性。
- **Mcp/McpEnv/McpUser + Trigger/TriggerExecution**：分别服务**工具目录、环境模板、用户按需安装，以及定时自动化**——都是"触达外部世界"的能力，集中管在 server 以便多端复用同一套 MCP 连接与调度游标。
- **Memory(Space/Project)**：服务**跨任务知识积累**，与 Memory 模块对齐——空间级共享、会话级隔离。
- **Config**：服务**工具凭据的统一存放与激活校验**，让 trigger/toolkit 共用同一份 ConfigGroup 凭据。
- **RemoteControl**：服务**远程设备接管（Brain 模式）**，`expires_at`/`bridge_status` 反映会话心搏。

一句话：**关系骨架承载"谁拥有什么"（租户/容器/作用域），JSON 载荷承载"它长得怎么样"（灵活扩展），Alembic 迁移承载"它怎么随产品长大"（含程序化数据搬移）**——三者共同支撑"控制面管一切持久状态"这一设计哲学。

---

## 迁移机制确认结论

通过程序化线索确认（grep `create_all/alembic/migrate` + 读 `alembic.ini`/`env.py`/`versions/`）：

- **机制 = Alembic**。`server/alembic.ini` 存在（`sqlalchemy.url` 默认 `postgresql://...eigent`）、`server/alembic/env.py` 用 `SQLModel.metadata` + `auto_import` 注册全部 model 包、`server/alembic/versions/` 有 10 个迁移文件，`down_revision` 串成一条链。
- **执行时机 = 启动脚本/容器，非应用内**。`server/start.sh` 先 `uv run alembic upgrade head` 再启 uvicorn；Dockerfile `CMD ["/app/start.sh"]`。运行时代码**不做 create_all**。
- **演进安全**：① `include_object` 忽略所有 `foreign_key_constraint`（外键演进不自动管）；② JSON blob + 提升列的渐进式加字段；③ space 层迁移内含**程序化数据回填 + 冲突消解**（legacy→space 折叠），全部 `ON CONFLICT DO NOTHING` 幂等；④ 每个迁移有对称 `downgrade()`。

## 验证与遗留

- **覆盖**：model/ 目录核心实体（Space/Project/User/ChatHistory/ChatStep/Mcp/McpUser/Trigger/TriggerExecution/Memory/SpaceFileIndex）均直接读到类定义，核心覆盖率 ≥ 90%；配置/迁移约 70%（`alembic.ini`/`env.py`/初始 schema/space 层迁移/trigger 迁移读毕，其余 7 个迁移由链条反推）。所有 `[路径:行号]` 经真源重定位校验。
- **已确认的边界**：
  - `space` 层迁移使用 `json_build_object`（PostgreSQL 专属函数，[server/alembic/versions/2026_05_25_1200-add_space_layer_foundation.py:304]()`），故 **space 层迁移无法在 SQLite 执行**——"SQLite 为切 URL 即可的兼容路径"仅在早期 schema 成立，space 之后的演进绑定 PostgreSQL。
  - 枚举序列化不统一是演进遗留：`Trigger` 用 `ChoiceType(..., String)`（[server/app/model/trigger/trigger.py:36]()），而 `Mcp`/`ChatHistory`/`User` 用 `SmallInteger`——字符串可读但占位更大，属"新增模块自带风格"的技术债。
- **遗留未决项**（供复核）：
  - `ChatSnapshot` 模型只有 `api_task_id/camel_task_id/browser_url/image_path/storage_key`，迁移却只给它加了 `storage_key`，没有 space_id/project_id/run_id——与 chat_history/chat_step/trigger 不同步，待确认是有意遗漏还是演进不一致。
  - PostgreSQL 连接池 `pool_size=36` 是否为实测调优值，或仅凭 docker 场景拍板（部署侧）。
  - `user_credits.py`（仅 `Channel(IntEnum)` 枚举）与 `user_credits_record.py`（`UserCreditsRecord` 表实体）命名易误导，需确认是否曾有独立表被重构合并。

