# 接入：MCP 端点、CLI、SDK 与 Agent 集成

> 本解读按 **database ⊗ ai-infra ⊗ ai-agent ⊗ framework** 类规格产出。本文是 OpenViking 的「接入/集成」关注面：一个上下文数据库要让各种 AI Agent 用起来，得暴露哪些口子、怎么把 Agent 的"历史"喂进来。核心是 **MCP 端点（读）、CLI/SDK（用户操作）、ingest 回灌（Agent 历史入库）** 三条接入通路。

## 接入心智：让 Agent 用文件系统的原语

OpenViking 的接入哲学与其存储哲学一脉相承：Agent 不通过"查询向量库"的黑盒 API 接入，而是通过一套**文件系统原语**（find / search / read / write / edit / grep / glob / tree / remember / forget）。这套原语以 MCP 工具的形式暴露给任何 MCP 客户端，也以 CLI/SDK 形式暴露给脚本和人类用户。

## MCP 端点：把 viking:// 变成一堆工具

server 暴露一个**流式 HTTP 的 MCP 端点** `/mcp`，用 `FastMCP` 注册 15 个工具——它就是"viking:// 文件系统"对外的标准协议面。会话生命周期绑定在 FastAPI lifespan（而非子应用 lifespan），保证请求到达前 task group 已就绪：

[openviking/server/app.py:351-354]() 在应用 lifespan 内 `mcp_lifespan()` 启动 MCP session manager，再接受 `/mcp` 请求。

<!-- anchor:         # Start MCP session manager (must be active before /mcp requests) …         async with mcp_lifespan(): -->

工具名在 `mcp_endpoint.py` 用 `@mcp.tool` 注册（是权威来源，见 `openviking/server/mcp_endpoint.py:7` 的注释与 15 处 `@mcp.tool()`）：

| MCP 工具 | 职责 |
|---|---|
| `find` / `search` | 语义检索：`find` 无会话语境，`search` 带会话意图 |
| `read` / `list` / `tree` | 文件系统浏览：读取/列目录/树形浏览 |
| `write` / `edit` | 内容写入/编辑（viking:// 路径） |
| `remember` | 写入记忆（Session 记忆接口） |
| `grep` / `glob` | 内容检索/通配匹配 |
| `add_resource` | 接入外部资源（git/url 等） |
| `list_watches` / `cancel_watch` | watch 管理 |
| `forget` | 删除 |
| `health` | 健康检查 |

<!-- anchor: async def find( … async def health() -> str: -->

> **为什么几乎整套文件系统原语都要进 MCP？** 因为 MCP 是当下编码 Agent（Claude Code、Codex、OpenCode、Cline 等）接入外部记忆的**最标准途径**。OpenViking 把"上下文浏览"降维成一组极像 `ls`/`grep`/`find` 的工具，让任何 MCP 客户端零认知成本就能用——这正是"Agent 用 ls/tree/find 浏览自己的上下文"在协议层的落地。

MCP 端点的消息传输走 streamable HTTP，身份经 HTTP scope 的 `X-OpenViking-Account` / `X-OpenViking-User` 头透传进 contextvars，`_extract_api_key` / `resolve_identity` 解析（`openviking/server/mcp_endpoint.py:15-20`）。

## 客户端 CLI（ov）与多语言 SDK

- **CLI**：`openviking_cli/` 是主要客户端实现，含 `client/`（HTTP 客户端）、`retrieve/`、`session/`、`utils/`、`setup_wizard.py`；`openviking` 包的 `client` 只是**惰性转发**到 `openviking_cli.client` 的兼容导出层。CLI 支持 `ov status / add-resource / ls / tree / find / grep / config` 全套文件系统操作。
- **Rust CLI**：`crates/ov_cli` 提供 Rust 实现，`openviking_cli/rust_cli.py` 可切换到 Rust 后端。

多语言 SDK 在 `sdk/` 下，三套齐备：

| SDK | 形态 |
|---|---|
| `sdk/go/` | Go 客户端（admin / client / filesystem / pack / helpers / errors） |
| `sdk/python/` | Python SDK 独立包（`openviking_sdk`） |
| `sdk/typescript/` | TypeScript 客户端（独立 TS 工程） |

## ingest：Agent 历史日志回灌成会话

OpenViking 让 Agent 的**历史会话**变成可检索的记忆，靠 `openviking/ingest/` 的"会话日志回灌"管线——解析各 agent harness 的本地会话日志，归一化后回灌进 OpenViking 会话并触发提交（commit 才产生记忆抽取）。

### 插件化接入新 harness

`registry.py` 用**一个装饰器注册新源**：

[openviking/ingest/registry.py:24-34]() `@register_source(name)` 把 `LogSource` 子类登记进 `SOURCE_REGISTRY`；docstring 明言"加一个 harness = 一个装饰器，无需改配置 schema"。这套机制让支持新 agent 的成本降到最低。

<!-- anchor: def register_source(name: str): …         return cls -->

### 已内置的源

`sources/` 下内置 6 个 harness + 基类：

| 源文件 | 对应 Agent | 数据形态 |
|---|---|---|
| `sources/claude_code.py` | Claude Code | JSONL |
| `sources/codex.py` | Codex | JSONL |
| `sources/opencode.py` | OpenCode | SQLite |
| `sources/hermes.py` | Hermes | JSONL |
| `sources/openclaw.py` | OpenClaw | JSONL |
| `sources/cursor.py` | Cursor | JSONL |
| `sources/base.py` | 基类 | `JsonlLogSource`（byte-offset 游标）/ `SqliteLogSource`（rowid 游标） |

### 回灌链路（幂等追加）

`IngestOrchestrator` 流式回灌：discovery → (reset/reconcile) → 游标增量读 → `set_pending` → append → commit：

[openviking/ingest/replay.py:3-16]() 模块 docstring 给出规范配方（幂等 batch）：每 ≤100 消息 batch 执行 `set_pending → append → confirm → commit_if_needed`；intent（目标游标 + size + 服务器先前消息数）在 append **前**持久化，崩溃后 `reconcile` 比对服务器当前消息数与基准判定 batch 是否落地——确认则不重放、丢弃则从确认游标重读。

回灌是**幂等追加**的核心场景：每 batch 先在本地记 `pending`（目标游标 + size + baseline），`append` 成功后 `confirm_append`；仅消费空消息时不欠 commit 只推进游标。

> **为什么回灌要"幂等 + 游标驱动"？** Agent 的会话日志往往很早就写完、但 OpenViking 可能是后装/初始化时一次性吞入。游标（byte-offset / rowid）保证增量续读，`set_pending`→`append`→`confirm` 保证崩溃重启不重不漏。这是"把外部已经很成熟的 Agent 历史导入自己系统"所需的工程严谨性。

## VikingBot：内建的 Agent 网关

`bot/` 的 VikingBot 是随服务可选启动的**多 IM 网关 Agent**，打通 Feishu/Slack/Telegram 等渠道：

- 通过 `viking://~` 家庭别名读写调用者自己的上下文空间（要求 server 支持 `viking://~`）。
- 能力：多 IM 入口、Agent 工具（file/shell/web/图像/定时/OpenViking 工具）、Skill 与 subagent、长期上下文（recall Resource/Memory/Experience + 自动提交会话）、Safety backends（sandbox）。
- Gateway 提供同步 chat、SSE 流式、feedback、OpenViking API 代理；`server/config.py:343` 配置 `bot_api_url = http://localhost:18790`。

## 集成生态

- `integrations/langchain`：提供 LangChain / LangGraph 集成。
- 各类直接 agent 集成（Claude Code / Codex / OpenClaw / Hermes / Cursor / TRAE / OpenCode / pi 等）把 OpenViking recall 注入 Agent 上下文，并自动提交会话记忆（见 README「Use it with your agent」清单）。

## 接入通路小结

| 通路 | 面向 | 承载 | 关键文件 |
|---|---|---|---|
| MCP 端点 /mcp | 任意 MCP 客户端 | 15 个文件系统原语工具（流式 HTTP） | `openviking/server/mcp_endpoint.py` |
| CLI `ov` + Rust CLI | 用户脚本/终端 | ls/tree/find/grep/add-resource/config | `openviking_cli/` |
| 多语言 SDK | 开发者 | go / python / typescript 三套客户端 | `sdk/` |
| ingest 回灌 | 已有 Agent 历史 | 6 harness JSONL/SQLite → 会话 → commit | `openviking/ingest/` |
| VikingBot | 端上使用者 | 多 IM 网关 Agent + skill + subagent | `bot/` |

> **三条通路的分工**：MCP 是**标准协议正门**（Agent 用），CLI/SDK 是**幂等操作口**（脚本/开发者用），ingest 是**历史导入泵**（把存量会话吸进来）。三者共用 viking:// 命名空间 + find/search/read/write 语义，真正做到"一处设计，处处一致"。
