# Vanna 2.0 — 生产级 NL→SQL Agent 框架

> 项目地址：https://github.com/vanna-ai/vanna  
> 许可证：MIT | 语言：Python >= 3.9 | 版本：2.0.2  
> 作者：Zain Hoda (zain@vanna.ai)

---

## 第一部分：快速概览

### 是什么

Vanna 是一个将自然语言转换为 SQL 并返回数据洞察的 Python 框架。核心理念是 **NL → SQL → Answers**。

2.0 版本从 0.x 时代的单一 `VannaBase` 类完全重写为现代化、模块化的 **Agent 架构**，引入了全链路用户感知、流式富组件 UI、7 大扩展点、30+ 集成等企业级能力。

### 解决什么痛点

| 痛点 | Vanna 的解法 |
|------|-------------|
| 业务用户不会写 SQL | 自然语言直接提问，LLM 自动生成 SQL |
| 数据查询缺乏安全控制 | 全链路用户感知 + 行级安全（RLS）+ 工具权限过滤 |
| 多租户场景权限混乱 | 基于用户组的工具级和 UI 级权限控制 |
| 数据分析结果呈现单一 | 流式富组件体系（表格、图表、卡片、进度追踪） |
| 企业合规审查困难 | 审计日志追踪每次查询和工具调用 |
| 前端开发成本高 | 预制 Web Component `<vanna-chat>`，一行代码嵌入 |
| Agent 行为不可控 | 7 大扩展点：生命周期钩子、错误恢复策略、中间件 |
| 从 0.x 迁移阻力大 | `LegacyVannaAdapter` 零摩擦兼容旧版 |

### 适用场景

- **嵌入式数据分析面板** — 在 SaaS 产品中嵌入 NL→SQL 问答界面
- **企业内部 BI 工具** — 为非技术团队提供自然语言数据查询能力
- **多租户数据平台** — 需要细粒度用户权限隔离的分析平台
- **AI Agent 原型开发** — 快速搭建工具增强的 LLM Agent 系统
- **数据 Copilot** — 为数据库配备 AI 助手（类 Copilot for Data）

### 核心能力

| 能力 | 说明 |
|------|------|
| NL→SQL 生成 | 支持 15+ LLM 提供商，自动生成并优化 SQL |
| 多数据库支持 | 15+ 数据库：PostgreSQL、MySQL、Snowflake、BigQuery、DuckDB 等 |
| 全链路用户感知 | 身份贯穿系统提示、工具权限、SQL 行级过滤、UI 可见性 |
| 流式富组件 UI | 表格、图表、卡片、任务列表、进度条 — 逐步流式渲染 |
| 预制 Web UI | `<vanna-chat>` Lit Web Component，框架无关 |
| 工具系统 | 内置 RunSQL、可视化、文件系统、Python 执行、记忆工具 |
| Agent 记忆 | 基于向量数据库的 few-shot 学习，存储工具调用和文本记忆 |
| 评估框架 | Trajectory/Output/LLM-as-Judge/Efficiency 四种评估器 |
| 审计日志 | 工具调用、AI 响应、用户操作全量审计 |
| 兼容 0.x | LegacyVannaAdapter 无缝迁移，保留自定义训练数据 |
| 双轨渲染 | 同时支持 RichComponent 和 SimpleComponent 两套 UI |

### 技术栈一览

| 层面 | 技术 |
|------|------|
| **语言** | Python >= 3.9 |
| **构建** | Flit (pyproject.toml) |
| **核心建模** | Pydantic >= 2.0 |
| **数据库抽象** | SQLAlchemy（核心）+ 各数据库原生驱动（可选） |
| **数据 & 图表** | Pandas, Plotly, tabulate, sqlparse |
| **HTTP** | httpx, requests |
| **CLI** | Click |
| **配置** | PyYAML |
| **可选服务器** | FastAPI >= 0.68 / Flask >= 2.0 |
| **前端** | Lit 3.x (Web Component), Vite, TypeScript, Storybook |
| **LLM 提供商** | OpenAI, Anthropic, Gemini, Ollama, Mistral, vLLM, Bedrock, 百度千帆, ZhipuAI |
| **向量存储** | ChromaDB, Qdrant, Pinecone, Milvus, FAISS, pgvector, Weaviate |
| **代码质量** | Ruff, Black, mypy |
| **测试** | pytest, pytest-asyncio, tox |

---

## 第二部分：架构深读

### 架构总览

```
┌────────────────────────────────────────────────────────────────────┐
│                    前端层 (Web Component)                           │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  <vanna-chat>  ───  ComponentManager                     │     │
│  │  SSE / WebSocket / Polling                               │     │
│  └──────────────────────────────────────────────────────────┘     │
└──────────────────────────┬───────────────────────────────────────┘
                           │ HTTP/SSE/WS
┌──────────────────────────▼───────────────────────────────────────┐
│                    服务器层 (FastAPI / Flask)                      │
│  ┌──────────────────────────────────────────────────────────┐     │
│  │  ChatHandler.handle_stream() → ChatStreamChunk 序列化     │     │
│  │  SSE 端点 → 流式 JSON 行                                  │     │
│  └──────────────────────────────────────────────────────────┘     │
└──────────────────────────┬───────────────────────────────────────┘
                           │
┌──────────────────────────▼───────────────────────────────────────┐
│                    核心层 — Agent                                 │
│                                                                   │
│  ┌─────────────────────┬──────────────────────┬────────────────┐ │
│  │  身份与权限          │  LLM 交互            │  会话管理       │ │
│  │  UserResolver       │  LlmService          │  Conversation  │ │
│  │  ToolRegistry       │  LlmMiddleware[]     │  Store         │ │
│  │  (权限过滤)          │  SystemPromptBuilder │  Filters[]     │ │
│  └─────────────────────┴──────────────────────┴────────────────┘ │
│  ┌─────────────────────┬──────────────────────┬────────────────┐ │
│  │  扩展点              │  上下文增强          │  可观测性       │ │
│  │  LifecycleHook[]    │  ToolContextEnricher │  Observability │ │
│  │  ErrorRecovery      │  LlmContextEnhancer  │  AuditLogger   │ │
│  │  WorkflowHandler    │  AgentMemory         │                │ │
│  └─────────────────────┴──────────────────────┴────────────────┘ │
└──────────┬──────────────────┬───────────────────┬────────────────┘
           │                  │                   │
           ▼                  ▼                   ▼
     ┌──────────┐    ┌──────────────┐    ┌──────────────┐
     │ LLM 集成 │    │ 工具系统      │    │ Agent Memory │
     │ OpenAI   │    │ RunSqlTool   │    │ 向量存储     │
     │ Anthropic│    │ Visualize    │    │ Few-shot     │
     │ Ollama   │    │ FileSystem   │    │ 文本记忆     │
     │ ... 15+  │    │ Python Exec  │    │ (10+ 向量DB) │
     └──────────┘    └──────┬───────┘    └──────────────┘
                            │
                            ▼
                     ┌──────────────┐
                     │ SQL Runner   │
                     │ (15+ 数据库)  │
                     └──────────────┘
```

### 启动流程

```
用户调用 vanna.create_agent()
  │
  ├─ 1. 创建 LlmService（绑定 LLM 提供商）
  ├─ 2. 创建 SqlRunner（绑定数据库）
  ├─ 3. 创建 ToolRegistry
  │    ├─ register_local_tool(RunSqlTool(sql_runner), access_groups=["user", "admin"])
  │    ├─ register_local_tool(VisualizeDataTool(...))
  │    └─ register_local_tool(create_file_system_tools(...))
  ├─ 4. 创建 UserResolver（绑定认证系统）
  ├─ 5. 创建 AgentMemory（绑定向量存储）
  ├─ 6. 创建 ConversationStore（绑定持久化存储）
  ├─ 7. 组装 Agent(llm_service, tool_registry, user_resolver, agent_memory, ...)
  ├─ 8. 创建 ChatHandler(agent)
  └─ 9. FastAPI: register_chat_routes(app, chat_handler)
       └─ /chat SSE 端点就绪
```

### 核心请求链路

```
用户: "2024年各季度营收趋势如何?"
  │
  ├─ 1. <vanna-chat> → SSE → ChatHandler.handle_stream()
  ├─ 2. UserResolver → User(id, group_memberships)
  ├─ 3. LifecycleHook.before_message() — 配额/限流检查
  ├─ 4. ConversationStore — 加载历史 + 添加消息
  ├─ 5. WorkflowHandler.try_handle() — 短路命令检查
  ├─ 6. ToolRegistry.get_schemas(user) — 按权限过滤工具
  ├─ 7. 构建系统提示 + LlmContextEnhancer（注入 few-shot 案例）
  ├─ 8. LLM 工具循环 (max_tool_iterations=10):
  │    ├─ 8a. LLM → run_sql(sql="SELECT ...")
  │    ├─ 8b. ToolRegistry.transform_args() — RLS 注入行级过滤
  │    ├─ 8c. LifecycleHook.before_tool()
  │    ├─ 8d. AuditLogger.log_tool_invocation()
  │    ├─ 8e. RunSqlTool.execute() → ToolResult + DataFrameComponent
  │    ├─ 8f. UI 组件流式推送 → 前端渲染表格
  │    ├─ 8g. LLM → visualize_data(...)  → ChartComponent
  │    └─ 8h. UI 组件流式推送 → 前端渲染 Plotly 图表
  ├─ 9. LifecycleHook.after_message()
  ├─10. AuditLogger.log_ai_response()
  └─11. SSE 关闭
```

### 模块职责划分

| 模块 | 路径 | 职责 |
|------|------|------|
| **Agent** | `core/agent/agent.py` | 编排 LLM 交互、工具执行、会话管理的核心入口 |
| **AgentConfig** | `core/agent/config.py` | 配置 Agent 行为：工具迭代次数、流式开关、UI 功能权限 |
| **Tool** | `core/tool/base.py` | 工具抽象基类：定义名称、描述、参数 schema、执行 |
| **ToolRegistry** | `core/registry.py` | 工具注册与权限管理：按用户组过滤、参数变换、执行流水线 |
| **LlmService** | `core/llm/` | LLM 服务抽象：封装请求/响应、流式/非流式 |
| **UserResolver** | `core/user/resolver.py` | 用户身份解析：从请求上下文提取用户信息与权限组 |
| **ConversationStore** | `core/storage/` | 会话存储抽象：消息持久化、历史加载 |
| **WorkflowHandler** | `core/workflow/` | 工作流处理器：LLM 前拦截，处理确定性场景 |
| **SystemPromptBuilder** | `core/system_prompt/` | 系统提示构建：根据用户和可用工具动态生成提示词 |
| **LifecycleHook** | `core/lifecycle/` | 生命周期钩子：消息前/后、工具前/后 |
| **LlmMiddleware** | `core/middleware/` | LLM 中间件：请求/响应拦截（缓存、成本追踪） |
| **ErrorRecovery** | `core/recovery/` | 错误恢复策略：重试、降级、跳过 |
| **ContextEnricher** | `core/enricher/` | 工具上下文增强：向 ToolContext 注入额外数据 |
| **LlmContextEnhancer** | `core/enhancer/` | LLM 上下文增强：增强系统提示（RAG、记忆注入） |
| **ConversationFilter** | `core/filter/` | 对话过滤器：发送给 LLM 前过滤/裁剪历史 |
| **Observability** | `core/observability/` | 可观测性：Span 追踪、指标记录 |
| **AuditLogger** | `core/audit/` | 审计日志：工具访问、调用、结果、AI 响应全记录 |
| **Evaluation** | `core/evaluation/` | 评估框架：Trajectory/Output/LLM-as-Judge |
| **UiComponent** | `core/components.py` | 前后端 UI 组件通信协议 |
| **ChatHandler** | `servers/base/chat_handler.py` | 框架无关的聊天处理逻辑 |
| **FastAPI/Flask** | `servers/fastapi/`, `servers/flask/` | 服务器路由注册 |

### 集成体系

Vanna 的集成分为四大类，全部通过 **optional extras** 按需安装：

**LLM 集成（15+）：** `openai`, `anthropic`, `gemini`, `ollama`, `mistralai`, `vllm`, `bedrock`, `azureopenai`, `qianfan`, `zhipuai`, `marqo`, `xinference-client`

**数据库集成（15+）：** `postgres`, `mysql`, `snowflake`, `bigquery`, `duckdb`, `clickhouse`, `oracle`, `mssql`, `hive`, `presto`, `sqlite`（内置）

**向量存储集成（10+）：** `chromadb`, `qdrant`, `pinecone`, `milvus`, `weaviate`, `faiss-cpu/gpu`, `opensearch`, `azuresearch`, `pgvector`, `marqo`

**服务器集成：** `fastapi` + `uvicorn`, `flask` + `flask-cors`

每个集成作为独立的 Python 子包（`integrations/<name>/`），遵循统一的接口抽象，通过依赖注入挂载到 Agent 中。

---

## 第三部分：Agent 核心设计

### 7 大扩展点

Vanna 2.0 最值得称道的架构决策是明确定义了 **7 个可扩展接口**，覆盖了 Agent 生命周期的每个关键节点（`src/vanna/core/agent/agent.py:56-80`）：

| 扩展点 | 接口 | 触发时机 | 典型用途 |
|--------|------|---------|---------|
| **LifecycleHook** | `before_message`, `after_message`, `before_tool`, `after_tool` | 消息和工具的前/后 | 配额检查、速率限制、自定义日志 |
| **LlmMiddleware** | `before_llm_request`, `after_llm_response` | LLM 请求/响应 | 请求缓存、成本追踪、响应脱敏 |
| **ErrorRecoveryStrategy** | `handle_tool_error`, `handle_llm_error` | 工具/LLM 执行失败 | 重试（指数退避）、降级回复、跳过 |
| **ToolContextEnricher** | `enrich_context` | 工具执行前 | 向 ToolContext 注入用户元数据、权限标签 |
| **LlmContextEnhancer** | `enhance_system_prompt`, `enhance_user_messages` | 构建 LLM 请求前 | RAG 注入、few-shot 记忆检索 |
| **ConversationFilter** | `filter_messages` | 发送 LLM 前 | 裁剪过长历史、移除敏感消息 |
| **ObservabilityProvider** | `record_metric`, `create_span`, `end_span` | 各关键节点 | Span 追踪、指标暴露给 Prometheus |

这些扩展点通过 **组合模式** 注入 Agent，而非继承：

```python
Agent(
    lifecycle_hooks=[RateLimitHook(), QuotaCheckHook()],
    llm_middlewares=[CacheMiddleware(), CostTrackingMiddleware()],
    error_recovery_strategy=RetryWithBackoffStrategy(),
    observability_provider=PrometheusObservabilityProvider(),
)
```

### Tool 抽象设计

Tools 是 Agent 能力的原子单元。每个 Tool 都遵循 `Tool[T]` 泛型接口（`src/vanna/core/tool/base.py:16-70`）：

```python
class Tool(ABC, Generic[T]):
    @property
    @abstractmethod
    def name(self) -> str: ...
    @property
    @abstractmethod
    def description(self) -> str: ...
    @property
    def access_groups(self) -> List[str]: return []
    @abstractmethod
    def get_args_schema(self) -> Type[T]: ...
    @abstractmethod
    async def execute(self, context: ToolContext, args: T) -> ToolResult: ...
    def get_schema(self) -> ToolSchema: ...
```

关键设计要点：

- **参数类型安全**：`get_args_schema()` 返回 Pydantic model，自动生成 JSON Schema 供 LLM 消费
- **权限声明**：`access_groups` 属性声明该工具所需的用户组
- **上下文注入**：`ToolContext` 携带用户身份、会话 ID、AgentMemory 引用
- **双通道输出**：`ToolResult` 同时包含 `result_for_llm` 和 `ui_component`

内置工具清单：

| 工具 | 类名 | 权限组 |
|------|------|--------|
| RunSQL | `RunSqlTool` | user, admin |
| 可视化 | `VisualizeDataTool` | user, admin |
| 文件操作 | `SearchFilesTool` / `ListFilesTool` / `ReadFileTool` / `WriteFileTool` / `EditFileTool` | user |
| Python 执行 | `RunPythonFileTool` / `PipInstallTool` | user |
| 记忆保存/搜索 | `SaveQuestionToolArgsTool` / `SearchSavedCorrectToolUsesTool` / `SaveTextMemoryTool` | admin |

### 权限机制

**第一层：工具可见性**

`ToolRegistry.get_schemas(user)`（`src/vanna/core/registry.py:90`）根据用户 `group_memberships` 过滤工具列表，只有用户组与工具 `access_groups` 有交集时，该工具的 Schema 才会暴露给 LLM。这意味着 **LLM 根本不知道无权工具的存在**。

**第二层：参数变换 (transform_args)**

`ToolRegistry.transform_args()`（`src/vanna/core/registry.py:113`）是一个 NoOp 方法，设计为子类覆盖。这是实现 **行级安全（RLS）** 的关键扩展点。

**第三层：UI 功能可见性**

`AgentConfig.ui_features.feature_group_access`（`src/vanna/core/agent/config.py:35-82`）控制前端 UI 功能的可见性。

---

## 第四部分：UI 组件体系

### Dual Render 设计

Vanna 的核心组件体系围绕 **双轨渲染** 设计（`src/vanna/core/components.py`）：

```python
class UiComponent(BaseModel):
    rich_component: RichComponent     # → 全功能前端
    simple_component: SimpleComponent # → 文本/图片降级
```

每个 `UiComponent` 同时携带两种表现形式，同一后端可以同时服务 Web Component 和纯 API 调用。

### 富组件类型

`ComponentType` 枚举定义了 24 种富组件类型（`src/vanna/core/rich_component.py:19-60`），包括文字（RichTextComponent）、数据表格（DataFrameComponent）、图表（ChartComponent）、状态卡片、进度条、按钮、任务列表、通知、卡片容器、工件展示（代码块/HTML/SVG）、日志查看器等。

### 组件生命周期

每个组件带有 `ComponentLifecycle` 指令（`src/vanna/core/rich_component.py:63-69`）：CREATE（创建）、UPDATE（更新）、REPLACE（替换）、REMOVE（移除）。前端 `ComponentManager` 维护组件状态树，支持流式逐步构建 UI。

### 简易集成

```html
<script src="https://img.vanna.ai/vanna-components.js"></script>
<vanna-chat sse-endpoint="https://your-api.com/chat" theme="dark"></vanna-chat>
```

支持 SSE、WebSocket、Polling 三种传输方式，自动降级。

---

## 第五部分：安全与多租户

### 全链路用户感知

从请求进入到 SQL 执行，每一个环节都能感知当前用户：`ChatHandler` 解析 `RequestContext` → `UserResolver.resolve()` → 系统提示注入用户信息 → `ToolRegistry.get_schemas(user)` 过滤工具 → `transform_args()` 注入 RLS → `AgentMemory` 按用户隔离记忆 → `UiFeatures` 控制 UI 可见性 → `AuditLogger` 记录用户上下文。

### 行级安全 (Row-Level Security)

通过 `ToolRegistry.transform_args()` 实现（`src/vanna/core/registry.py:113`）。框架提供插槽，使用者注入策略：

```python
class MyToolRegistry(ToolRegistry):
    async def transform_args(self, tool, args, user, context):
        if tool.name == "run_sql":
            if "WHERE" in args.sql.upper():
                args.sql = args.sql.replace("WHERE",
                    f"WHERE org_id = '{user.metadata['org_id']}' AND ")
            else:
                args.sql = args.sql.rstrip(";") + f" WHERE org_id = '{user.metadata['org_id']}'"
        return args
```

### 审计日志

9 种审计事件类型（`src/vanna/core/audit/models.py:16-34`）：`TOOL_ACCESS_CHECK`、`TOOL_INVOCATION`、`TOOL_RESULT`、`UI_FEATURE_ACCESS_CHECK`、`AI_RESPONSE_GENERATED`、`MESSAGE_RECEIVED`、`CONVERSATION_CREATED`、`ACCESS_DENIED`、`AUTHENTICATION_ATTEMPT`。支持参数自动脱敏。

### 速率限制

通过 `LifecycleHook.before_message()` 实现，与核心 Agent 逻辑解耦。

---

## 第六部分：工作流与评估

### 短路工作流 (WorkflowHandler)

`WorkflowHandler`（`src/vanna/core/workflow/base.py:74-254`）是一种 **短路模式**，在 LLM 处理之前拦截消息。`try_handle()` 返回 `should_skip_llm=True` 则直接返回组件结果；`get_starter_ui()` 在会话开始时返回欢迎 UI。`DefaultWorkflowHandler` 实现了 `/help`、`/status`、`/memories`、`/delete <id>` 等命令。

### 评估框架

四种评估器（`src/vanna/core/evaluation/`）：

| 评估器 | 评估内容 |
|--------|---------|
| **TrajectoryEvaluator** | Agent 执行轨迹（工具调用序列） |
| **OutputEvaluator** | 最终输出质量（SQL 正确性） |
| **LlmAsJudgeEvaluator** | LLM 评判输出质量 |
| **EfficiencyEvaluator** | 执行效率（耗时、token 消耗） |

---

## 第七部分：兼容与迁移

### LegacyVannaAdapter

`LegacyVannaAdapter`（`src/vanna/legacy/adapter.py`）通过 **适配器模式** 将 0.x 的 `VannaBase` 实例包装为新的 `ToolRegistry` + `AgentMemory`。用户积累的以下资产全部保留：自定义训练数据、向量存储嵌入、已有的 `run_sql` 实现、自定义 `generate_sql` 逻辑。唯一的改动是将 `VannaBase` 实例传给 `LegacyVannaAdapter`，然后使用新 Agent 的 Web UI。

### 0.x → 2.0 架构对比

| 维度 | 0.x (VannaBase) | 2.0 (Agent) |
|------|-----------------|-------------|
| 架构模式 | 大基类 + 覆写方法 | 组合模式 + 依赖注入 |
| 可扩展性 | 覆写几个方法 | 7 大扩展点接口 |
| 用户感知 | 无 | 全链路用户对象传递 |
| UI | 无内置 UI | 流式富组件 + Lit Web Component |
| 传输 | 同步阻塞 | 异步流式 (SSE/WebSocket/Poll) |
| 权限 | 无 | 三层权限体系 |
| 审计 | 无 | 9 种审计事件类型 |
| 评估 | 无 | 4 种评估器 |

---

## 第八部分：同类对比

| 维度 | Vanna 2.0 | LangChain SQL Agent | Databricks Genie | Supabase AI |
|------|-----------|-------------------|-----------------|-------------|
| 定位 | 专用 NL→SQL Agent 框架 | 通用 Agent 框架的 SQL 组件 | 托管数据分析产品 | 托管数据库 AI 助手 |
| 部署方式 | 自建 Python 服务 | 自建 Python 服务 | SaaS 托管 | SaaS 托管 |
| UI | 内置 Lit Web Component | 无内置 UI | 内置 Web UI | 内置 Web UI |
| 流式渲染 | 富组件流式 | 文本流式 | 有限流式 | 有限流式 |
| 用户感知 | 全链路 | 需自行实现 | 内置 | 内置 |
| 权限粒度 | 工具级 + UI 级 + RLS 扩展点 | 无内置 | 工作区级 | 表级 RLS |
| 审计 | 9 种事件 | 无内置 | 内置 | 内置 |
| 评估框架 | 4 种评估器 | LangSmith 外部 | 内置 | 有限 |
| 多数据库 | 15+ | 有 | 仅 Databricks | 仅 Supabase |
| 许可证 | MIT 开源 | MIT 开源 | 商业闭源 | 部分开源 |

---

## 总结评价

### 最值得学习的设计决策

1. **组合优先于继承**：7 大扩展点全部通过组合注入，而非继承覆写。这是从 0.x 大基类模式的关键教训，使得 Agent 行为可以像搭积木一样自由组装。
2. **用户感知的层次化设计**：用户身份不是简单的一个 token，而是从 LLM 提示词、工具可见性、参数变换、UI 控制到审计日志的全链路贯穿，体现了真正的 end-to-end 安全思维。
3. **双通道输出 (ToolResult)**：同时为 LLM 和 UI 提供结果，避免了"先生成文本、再解析成 UI 组件"的繁琐模式，这是 Agent 框架设计与前端框架设计相结合的精妙产物。
4. **适配器模式的迁移策略**：`LegacyVannaAdapter` 让 0.x 用户可以零摩擦获得 2.0 的 UI 能力，同时保留自定义训练数据，这种渐进式迁移策略极大降低了用户的升级阻力。

### 局限性

1. **前端锁定**：虽然 Web Component 是框架无关的，但富组件体系的协议是 Vanna 自定的，而非 OpenAPI/AsyncAPI 等标准协议。
2. **RLS 示例代码中的 SQL 注入风险**：框架文档中的 RLS 示例使用 f-string 拼接 org_id，实际生产环境中应使用参数化查询或 SQLAlchemy 表达式。
3. **企业级 SSO 集成**：`UserResolver` 的接口定义非常通用，但缺少内置的 OIDC/SAML 集成实现，需要使用者自行实现。
4. **单机模型部署**：虽然支持 Ollama/vLLM，但对于需要 GPU 推理的大规模部署场景，缺少原生的模型路由和负载均衡支持。

---

> 项目地址：[https://github.com/vanna-ai/vanna](https://github.com/vanna-ai/vanna)
