# OpenWorker 核心引擎与 Agent 体系深度分析

## 1. TurnEngine — Agent 核心循环

**文件:** `coworker/engine.py`（~1192 行）

TurnEngine 是 OpenWorker 的心脏——一个异步 agent 循环引擎，负责协调用户输入、LLM 推理和工具执行。

### 1.1 循环模型

```
用户输入（turn 开始）
    │
    ▼
┌───────────────────────────────────────────┐
│  context_provider()                       │
│  ├─ 注入系统提示词（agent persona）        │
│  ├─ 注入对话历史                          │
│  ├─ 注入工具定义（当前可用的工具列表）      │
│  └─ 注入记忆/技能/环境上下文               │
└───────────────┬───────────────────────────┘
                ▼
┌───────────────────────────────────────────┐
│  provider.chat() ← LLM 推理               │
│  ├─ 流式输出文本 → 事件推送 surface       │
│  └─ 非流式获取完整响应                     │
└───────────────┬───────────────────────────┘
                ▼
    检查模型输出类型
    │           │
    文本回复     工具调用
    │           │
    │    ┌──────▼──────────────────────┐
    │    │  PermissionEngine 判断       │
    │    │  ├─ ALLOW → 执行工具         │
    │    │  │   ├─ 低风险 → 并发执行     │
    │    │  │   └─ 高风险 → 顺序执行     │
    │    │  ├─ NEEDS_USER → 发出审批事件 │
    │    │  └─ DENY → 拒绝并通知        │
    │    └──────┬──────────────────────┘
    │           │
    │    ┌──────▼──────┐
    │    │  工具执行完成  │
    │    │  结果注入上下文 │
    │    └──────┬──────┘
    │           │
    └────── 继续调用 LLM (goto step 2) ──┘
                │
         达到 max_iter 或 LLM 主动结束
                │
                ▼
         发送最终文本回复 + 完成事件
```

**关键特性：**

| 特性 | 实现方式 |
|------|---------|
| **并发读/顺序写** | 低风险工具并行执行（asyncio.gather）；写/Shell 串行排队 |
| **阻塞隔离** | Provider/Tool 阻塞调用通过 `asyncio.to_thread` 包装 |
| **审批注入** | 审批器以异步 `approver` 协程注入，等待用户响应 |
| **迭代控制** | `max_iter` 配置上限，防止无限循环 |
| **事件流** | 每次 LLM 输出和工具结果通过 Event 通道推送至 surface |

### 1.2 context_provider

在每次 LLM 调用前动态构建上下文，包含：

- **System Prompt**：从 Agent Persona 获取
- **Conversation History**：当前会话消息
- **Tool Definitions**：当前权限模式下可用的工具（PermissionEngine 过滤）
- **Memory Injunctions**：持久记忆中的相关条目
- **Skills**：当前任务相关的技能定义
- **Environment Context**：工作区路径、Shell 环境变量等
- **Override Hints**：用户设置的风险覆盖策略

---

## 2. Agent 体系

**目录:** `coworker/agents/`

OpenWorker 有四种 Agent，每种有独立的提示词、工具集和权限配置：

| Agent | 类文件 | 适用场景 | 需要工作区 | 可用工具 |
|-------|--------|---------|-----------|---------|
| **Code** | `agents/code.py` | 编程任务 | ✅ | code_files + git + search + shell + todo + explore subagent |
| **Cowork** | `agents/cowork.py` | 知识工作（文档/分析/调研） | ✅ | files（多根）+ search + shell + todo |
| **Chat** | `agents/chat.py` | 纯对话/问答 | ❌ | 无文件/shell 访问，仅记忆+技能 |
| **MyHelper** | `agents/myhelper.py` | 个人助手 | ✅ | 同 Cowork 工具集，不同提示词 |

### 2.1 Agent 装配流程

```
Agent 实例化
    │
    ├─ 创建 AgentContext
    │   ├─ workspace (工作区路径)
    │   ├─ executor (异步执行器)
    │   ├─ todo (待办管理)
    │   └─ roots (多根目录)
    │
    ├─ build_tools()
    │   └─ catalog.expand()
    │       ├─ 注册 Agent 自有工具
    │       ├─ 注册 Connector 工具
    │       ├─ 注册 MCP 工具
    │       ├─ 注册 Skill 工具
    │       └─ PermissionEngine 过滤
    │
    └─ 注册到 AgentRegistry
```

Agent 通过 `AgentContext` 接收所有依赖组件，`build_tools()` 调用工具目录 `catalog.expand()` 展开完整的工具列表并交由权限引擎过滤。

### 2.2 Persona（角色）系统

**目录:** `coworker/personas/`

Persona 是一等公民抽象——每个 Agent 绑定一个 Persona，Persona 定义了：

| 组件 | 说明 |
|------|------|
| System Prompt | 角色行为准则和专业知识 |
| Tool Set | 该角色可用的工具白名单 |
| Connector Config | 绑定的第三方服务配置 |
| Override Rules | 默认的风险覆盖策略 |

内置 Persona 示例：

```markdown
# Built-in Ops Persona (ops.md)
你是一名 DevOps 运维专家。你可以：
- 管理系统服务和进程
- 监控系统资源
- 排查网络和数据库问题
- 执行日常运维任务
```

### 2.3 Agent 注册表

**文件:** `agents/registry.py`

管理所有可用 Agent 类型，支持按名称查找和条件加载。注册表在 Server 启动时初始化，注入到 SessionManager 中供引擎使用。

---

## 3. 工具系统

### 3.1 内置工具注册表

**目录:** `coworker/tools/`

| 工具 | 文件 | 风险等级 | 说明 |
|------|------|---------|------|
| `file_read` | `tools/files.py` | 读 | 文件读取 |
| `file_write` | `tools/files.py` | 写 | 文件写入/替换/打补丁 |
| `shell_exec` | `tools/shell.py` | Shell | Shell 命令执行 |
| `grep_search` | `tools/search.py` | 读 | 内容搜索 |
| `file_search` | `tools/search.py` | 读 | 文件搜索（glob） |
| `git_ops` | `tools/git.py` | 写 | Git 操作（commit/push 等） |
| `todo_manage` | `tools/todo.py` | 读 | 待办清单管理 |
| `propose_plan` | `tools/plan.py` | 读 | 规划工具（propose → refine → commit） |
| `ask_user` | `tools/ask.py` | 读 | 向用户提问 |
| `explore` | `tools/subagent.py` | 读 | 子代理（并行代码探索） |
| `memory_tools` | `memory/tools.py` | 读写 | 持久记忆读写 |
| `connector_tools` | `connectors/tools.py` | 写 | 连接器消息发送 |

### 3.2 工具定义结构

每个工具注册时包含：

```python
@dataclass
class ToolDef:
    name: str              # 工具名称（LLM 调用用）
    description: str       # 描述
    parameters: dict       # JSON Schema 参数定义
    handler: Callable      # 执行函数
    risk_level: RiskLevel  # 风险等级（read/write/shell/send）
```

### 3.3 工具目录 (Catalog)

**文件:** `coworker/catalog.py`

Catalog 是所有"vetted capability"（经过审查的能力）的注册中心。`expand()` 方法根据：

1. **Agent 类型** — 不同的 Agent 可用的工具不同
2. **权限模式** — Discuss 模式下写工具被过滤
3. **角色绑定** — Persona 配置的工具白名单
4. **环境上下文** — 某些工具需要特定环境（如 git 需要 repo）

动态生成当前 turn 的工具列表。

---

## 4. 技能系统

**目录:** `coworker/skills/`

技能（Skill）是可重用的能力包（Anthropic `SKILL.md` 格式）。在 2026-08-01 的 PR #391 后，技能系统从"加载器"升级为完整的**加载 + 管理**两大部分：

| 部分 | 文件 | 职责 |
|------|------|------|
| **加载层** | `skills/base.py`（`SkillLoader`） | 渐进式暴露：会话开始只注入目录（name+description），按需通过 `load_skill` 加载完整正文（未命中会 `rescan` 兜底） |
| **管理层** | `skills/store.py`（`SkillStore` / `SessionSkillStore`） | 文件夹即真相的 CRUD（global/project 双作用域）、上传 stage→preview→confirm、会话级静音、worker 自建 `save_skill` 工具 |

引擎通过注入一个 `skill_filter` **可调用体**来获取当前会话的技能目录（`manager.py:467`），使 Settings 禁用与新建技能对**在线会话立即生效**。完整的加载、CRUD、上传、静音、启用优先级与代码索引详见：

> **→ [技能（Skills）系统深度分析](./skills-system.md)**

---

## 5. 记忆系统

**目录:** `coworker/memory/`

| 组件 | 说明 |
|------|------|
| `base.py` | MemoryStore 抽象基类 |
| `sqlite_store.py` | SQLite 持久化实现 |
| `tools.py` | 记忆工具（remember/forget/list/update） |

**双作用域：**

| 作用域 | 范围 | 生命周期 |
|--------|------|---------|
| GLOBAL | 所有会话共享 | 用户级持久 |
| WORKSPACE | 绑定到特定工作区 | 项目级持久 |

记忆条目支持结构化存储，每条记录包含内容、作用域、时间戳、关键程度等元数据。

---

## 6. 会话与引擎装配

### 6.1 SessionManager

**文件:** `coworker/server/manager.py`（~4086 行）

SessionManager 是 OpenWorker 中最大的模块，负责：

1. **会话 CRUD**：创建/激活/过期/清理会话
2. **引擎装配**：为每个会话调用 `build_engine()` 装配完整的 TurnEngine
3. **组件组装**：将 provider router、MCP manager、memory store、inbox、connectors、automation scheduler、agent registry 等注入引擎
4. **生命周期管理**：会话超时回收、闲置关闭

### 6.2 引擎装配 (`agent.py`)

```
build_engine()
    │
    ├─ 1. Resolve Agent → Agents.resolve(agent_type)
    │       └─ 获取 Agent 类 + Persona
    │
    ├─ 2. Build Provider → ProviderRouter.select(model_id)
    │       └─ 根据模型 ID 选择 ProviderClient 实例
    │
    ├─ 3. Build Tools → Catalog.expand(agent, permission_mode)
    │       └─ 展开 Agent 可用工具列表
    │
    ├─ 4. Build Memory → MemoryStore(session_id)
    │       └─ 创建/加载会话记忆存储
    │
    ├─ 5. Build Skills → SkillLoader(workspace)
    │       └─ 加载当前工作区的技能
    │
    ├─ 6. Build Connectors → ConnectorManager.list_active()
    │       └─ 获取已配置的连接器
    │
    └─ 7. Assemble TurnEngine(provider, tools, memory, ...)
```

---

## 7. 并发模型

| 场景 | 策略 | 实现 |
|------|------|------|
| 多工具并行（读/搜索） | 并发执行 | `asyncio.gather(*low_risk_tools)` |
| 单轮多工具（写/Shell） | 顺序执行 | 按调用顺序逐个 await |
| LLM 阻塞调用 | 线程池隔离 | `asyncio.to_thread` 包装同步调用 |
| 多会话 | 事件循环共享 | 每个 Session 有独立的 engine，共享 event loop |
| 审批等待 | 异步挂起 | `approver` 协程等待用户事件 |

---

## 8. 关键数字

| 参数 | 默认值 | 位置 |
|------|--------|------|
| Agent 循环 max_iter | 可配置（通常 20-50） | `engine.py` |
| 工具并发上限 | 读工具并发，写工具串行 | `engine.py` |
| Session 超时 | 可配置 | `manager.py` |
| 记忆作用域 | GLOBAL + WORKSPACE | `memory/base.py` |
| 技能来源 | 全局 + 工作区 | `skills/base.py` |
| MCP 传输 | stdio + streamable HTTP | `mcp/client.py` |
| 上下文压缩触发 | `min(0.8×window, 250k)` | `compaction.py` |

---

## 9. 代码索引

| 想了解什么 | 看哪个文件 |
|-----------|-----------|
| Agent 循环核心 | `coworker/engine.py` |
| 引擎装配入口 | `coworker/agent.py` |
| 会话管理 | `coworker/server/manager.py` |
| Code Agent | `coworker/agents/code.py` |
| Cowork Agent | `coworker/agents/cowork.py` |
| Chat Agent | `coworker/agents/chat.py` |
| MyHelper Agent | `coworker/agents/myhelper.py` |
| Agent 注册表 | `coworker/agents/registry.py` |
| Agent 基类 | `coworker/agents/base.py` |
| 文件工具 | `coworker/tools/files.py` |
| Shell 工具 | `coworker/tools/shell.py` |
| 搜索工具 | `coworker/tools/search.py` |
| Git 工具 | `coworker/tools/git.py` |
| 规划工具 | `coworker/tools/plan.py` |
| 子代理工具 | `coworker/tools/subagent.py` |
| 工具注册表 | `coworker/tools/registry.py` |
| 工具目录 | `coworker/catalog.py` |
| 记忆存储 | `coworker/memory/sqlite_store.py` |
| 记忆工具 | `coworker/memory/tools.py` |
| 技能加载器（渐进式暴露） | `coworker/skills/base.py` |
| 技能管理（CRUD/上传/静音） | `coworker/skills/store.py` |
| 角色清单 | `coworker/personas/manifest.py` |
| 角色注册表 | `coworker/personas/registry.py` |
| 角色加载 | `coworker/personas/loading.py` |
| FastAPI 应用 | `coworker/server/app.py` |
| 服务器启动 | `coworker/server/run.py` |
