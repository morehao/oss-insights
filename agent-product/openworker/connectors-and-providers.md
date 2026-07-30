# OpenWorker 连接器与 LLM 提供商体系深度分析

## 1. LLM Provider 体系

OpenWorker 的 LLM 接入层统一由 `aisuite` 框架驱动。Andrew Ng 团队在 aisuite 上构建了一层 `ProviderRouter`，实现模型前缀路由。

### 1.1 架构概览

```
LLM 调用入口 (TurnEngine)
    │
    ▼
┌──────────────────────────────────┐
│  ProviderRouter.select(model)    │  ← 按模型字符串前缀路由
│  "openai/gpt-4o" → OpenAI       │
│  "claude/sonnet" → Anthropic    │
│  "gemini/pro"   → Gemini        │
│  "vertex/..."   → Vertex AI     │
│  "bedrock/..."  → Bedrock       │
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  ProviderClient (抽象基类)        │
│  ├─ chat()        ← 流式/非流式  │
│  ├─ count_tokens() ← Token 计量  │
│  └─ supports_*()  ← 能力检测    │
└──────────────┬───────────────────┘
               │
     ┌─────────┼─────────┬──────────┐
     ▼         ▼         ▼          ▼
  OpenAI   Anthropic   Gemini    Bedrock
  Provider  Provider   Provider  Provider
```

### 1.2 Provider 实现

**目录:** `coworker/providers/`

| Provider | 文件 | 支持流式 | 提示缓存 | Token 计量 |
|----------|------|---------|---------|-----------|
| OpenAI | `openai_provider.py` | ✅ | ✅ | ✅ |
| Anthropic | `anthropic_provider.py` | ✅ | ✅ | ✅ |
| Gemini | `gemini_provider.py` | ✅ | ❌ | ✅ |
| Vertex AI | `vertex_provider.py` | ✅ | ❌ | ✅ |
| Bedrock | `bedrock_provider.py` | ✅ | ❌ | ✅ |

**统一 ProviderClient 抽象：**

```python
class ProviderClient(ABC):
    @abstractmethod
    async def chat(
        self,
        messages: list[Message],
        tools: list[ToolDef] | None = None,
        stream: bool = True,
    ) -> AsyncIterator[ChatEvent]:
        ...

    @abstractmethod
    def count_tokens(self, text: str) -> int:
        ...

    def supports_prompt_caching(self) -> bool:
        return False
```

### 1.3 ProviderRouter

**文件:** `providers/router.py`

路由规则基于模型字符串前缀：

| 前缀模式 | 目标 Provider |
|----------|-------------|
| `openai/` 或 `gpt-` | OpenAI |
| `claude/` 或 `claude-` | Anthropic |
| `gemini/`  | Gemini |
| `vertex/` 或 `vertex_` | Vertex AI |
| `bedrock/` | AWS Bedrock |
| `ollama/` | Ollama（本地） |

支持运行时动态切换 provider——在对话中更换 model 字符串即可切换底层 LLM。

### 1.4 Capabilities & Matrix

**文件:** `providers/capabilities.py`, `providers/matrix.py`

每个 Provider 注册其能力矩阵，包括：

- 最大上下文窗口
- 支持的 tools/functions 格式
- 流式支持类型
- prompt caching 支持
- 价格信息（用于 Token 计量）

Capabilities 系统在 ProviderRouter 选择时辅助决策，确保选用的模型满足任务需求。

---

## 2. 连接器体系

**目录:** `coworker/connectors/`

连接器是 OpenWorker 与外部服务的桥梁。25+ 连接器被组织为统一的 Adapter 模式。

### 2.1 架构概览

```
外部事件 (Slack/GitHub 等)
    │
    ▼
┌──────────────────────────────┐
│  Connector Adapter            │  ← 各自实现 ConnectorBase
│  ├─ handle_event()            │
│  └─ format_message()          │
└──────────┬───────────────────┘
           │ 标准化为内部事件
           ▼
┌──────────────────────────────┐
│  Gateway (消息网关)           │  ← 分发入站消息
│  ├─ route_incoming()          │
│  └─ dispatch_to_session()     │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│  Session Manager              │  ← 创建/路由到会话
└──────────┬───────────────────┘
           │
           ▼
       TurnEngine 处理
           │
           ▼
┌──────────────────────────────┐
│  Senders (出站发送器)          │  ← 回复消息
│  ├─ send_message()            │
│  └─ format_target()           │
└──────────┬───────────────────┘
           │
           ▼
     外部服务 (Slack/Telegram 等)
```

### 2.2 连接器分类

| 类别 | 连接器 | 文件前缀 |
|------|--------|---------|
| **通讯** | Slack, Telegram, Email | `slack_*`, `telegram_*`, `email_*` |
| **代码协作** | GitHub (Issues/PRs/Code Review) | `github_*` |
| **项目管理** | Jira, Linear, Monday.com | `jira_*`, `linear_*` |
| **日历** | Google Calendar, Outlook | `gcal_*` |
| **邮箱** | Gmail, Outlook Mail | `gmail_*`, `outlook_*` |
| **CRM** | HubSpot | `hubspot_*` |
| **文档** | Notion | `notion_*` |
| **浏览器** | Playwright 自动化 | `browser_automation.py` |
| **实验性** | Discord, WhatsApp 等 | `experimental/` |

### 2.3 ConnectorBase 抽象

```python
class ConnectorBase(ABC):
    """所有连接器的基类"""

    @property
    @abstractmethod
    def name(self) -> str: ...

    @abstractmethod
    async def handle_event(self, event: ExternalEvent) -> InternalMessage:
        """将外部事件转换为内部统一消息格式"""
        ...

    @abstractmethod
    async def send_message(self, target: str, content: MessageContent) -> None:
        """向外部服务发送消息"""
        ...

    async def health_check(self) -> bool:
        """健康检查"""
        ...
```

### 2.4 消息网关

**文件:** `connectors/gateway.py`

Gateway 是连接器与引擎之间的消息调度中枢：

- **入站路由**：`route_incoming()` 将外部消息分发到正确的 Session
- **出站分发**：`dispatch_outgoing()` 将引擎回复发送到对应的连接器
- **连接器发现**：自动发现已激活的连接器

### 2.5 认证与配置

**文件:** `connectors/config.py`, `connectors/setup.py`, `connectors/accounts.py`

连接器认证分为两种模式：

| 模式 | 说明 | 使用场景 |
|------|------|---------|
| **OAuth** | 通过 OpenWorker Cloud 中继的 OAuth 流程 | Slack, GitHub, Google |
| **Token/API Key** | 直接存储密钥 | Telegram Bot, Email IMAP |

密钥存储在 `coworker/secrets.py`（本地加密存储），连接器配置与帐户管理分离。

### 2.6 WebSocket 中继

**文件:** `connectors/relay_client.py`

OpenWorker Cloud 提供 WebSocket 中继服务（relay client），用于：

- **托管 OAuth**：云服务处理 OAuth 回调，通过中继推送 Token
- **事件中继**：接收外部服务的 webhook 事件
- **跨设备同步**：连接器状态在多个桌面间同步

### 2.7 Adapter 模式

**文件:** `connectors/adapters.py`

Adapter 负责将连接器的原生数据模型转换为引擎可理解的统一格式：

```
SlackMessage → InternalMessage
GitHub Issue → InternalMessage
Telegram Text → InternalMessage

InternalMessage 格式：
{
    "role": "user",
    "source": "slack/github/telegram/...",
    "content": "...",
    "attachments": [...],
    "metadata": {...}
}
```

---

## 3. MCP 协议支持

**目录:** `coworker/mcp/`

OpenWorker 内置完整的 MCP（Model Context Protocol）客户端实现。

### 3.1 传输支持

| 传输方式 | 文件 | 说明 |
|---------|------|------|
| **stdio** | `mcp/client.py` | 子进程 stdio 通信 |
| **streamable HTTP** | `mcp/client.py` | HTTP SSE 流式通信 |
| **OAuth** | `mcp/oauth.py` | MCP 服务 OAuth 认证 |

### 3.2 MCP 工具集成

MCP 工具通过以下流程集成到引擎：

```
1. MCP 服务器启动（stdio/HTTP）
2. mcp/client.py 初始化连接
3. 调用 list_tools() 获取工具定义
4. 工具定义注入 Tool Registry
5. PermissionEngine 对 MCP 工具单独控制权限
6. 引擎调用 MCP 工具 → 结果返回
```

每个 MCP 工具可以单独控制权限——管理员可以设置某些 MCP 工具需要审批，某些自动放行。

### 3.3 配置

**文件:** `mcp/config.py`

MCP 服务器配置支持：

```toml
[mcp.servers.my-server]
command = "python"
args = ["-m", "my_mcp_server"]
transport = "stdio"

[mcp.servers.cloud-api]
url = "https://api.example.com/mcp"
transport = "streamable-http"
auth = "oauth"
```

---

## 4. 连接器与管理工具

### 4.1 连接器工具

**文件:** `connectors/tools.py`

引擎通过 `send_message` 工具与外部服务交互：

```python
send_message(target: str, content: str, attachments: list = None)
```

`target` 参数格式为 `connector_id:channel_id`，由 Gateway 负责路由到正确的连接器。

### 4.2 连接器管理

**文件:** `connectors/catalog_copy.py`, `connectors/descriptors.py`

- `catalog_copy.py`：连接器目录的本地缓存
- `descriptors.py`：连接器描述符（名称、图标、认证方式等）
- `accounts.py`：连接器账号管理
- `parked.py`：待处理消息队列

### 4.3 浏览器自动化

**文件:** `connectors/browser_automation.py`

基于 Playwright 的浏览器自动化，用于：

- 无 API 的网页操作（登录、填表、截图）
- 网页信息抓取
- 端到端测试场景

---

## 5. 连接器关键数字

| 参数 | 数值 |
|------|------|
| 连接器总数 | 25+ |
| 核心通讯连接器 | Slack, Telegram, Email |
| 代码协作连接器 | GitHub (Issues/PRs/Code Review) |
| 项目管理连接器 | Jira, Linear, Monday.com |
| 日历连接器 | Google Calendar, Outlook |
| 邮箱连接器 | Gmail, Outlook Mail |
| MCP 传输方式 | stdio, streamable HTTP |
| Provider 数量 | 5 |

---

## 6. 代码索引

| 想了解什么 | 看哪个文件 |
|-----------|-----------|
| Provider 抽象基类 | `coworker/providers/base.py` |
| OpenAI Provider | `coworker/providers/openai_provider.py` |
| Anthropic Provider | `coworker/providers/anthropic_provider.py` |
| Gemini Provider | `coworker/providers/gemini_provider.py` |
| Vertex AI Provider | `coworker/providers/vertex_provider.py` |
| Bedrock Provider | `coworker/providers/bedrock_provider.py` |
| Provider 路由器 | `coworker/providers/router.py` |
| Provider 注册表 | `coworker/providers/registry.py` |
| 能力矩阵 | `coworker/providers/matrix.py` |
| 能力检测 | `coworker/providers/capabilities.py` |
| 连接器基类 | `coworker/connectors/base.py` |
| ADapter 模式 | `coworker/connectors/adapters.py` |
| 消息网关 | `coworker/connectors/gateway.py` |
| 消息发送器 | `coworker/connectors/senders.py` |
| 消息发送工具 | `coworker/connectors/tools.py` |
| WebSocket 中继 | `coworker/connectors/relay_client.py` |
| MCP 客户端 | `coworker/mcp/client.py` |
| MCP 配置 | `coworker/mcp/config.py` |
| MCP OAuth | `coworker/mcp/oauth.py` |
| MCP 工具 | `coworker/mcp/tools.py` |
| 连接器配置 | `coworker/connectors/config.py` |
| 连接器设置 | `coworker/connectors/setup.py` |
| 浏览器自动化 | `coworker/connectors/browser_automation.py` |
