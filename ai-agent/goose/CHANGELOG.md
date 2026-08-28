# 变更记录

> 本解读对 goose 的跨版本变更汇总；按版本倒序。描述清楚相关变动即可：若上游有官方 changelog，参照它并结合代码变动生成；没有则基于代码变动（git diff / release notes / 代码快照比对）生成。本次因本地源码为浅克隆（depth=1）无上游 git 历史，全部条目基于**代码快照比对**（当前 `main` 与既有解读描述的差异）生成，非官方 changelog。

## [main@f87c4d7（2026-08-28）]

- 更新日期：2026-08-28。依据：**代码快照比对**（当前 HEAD `f87c4d7` 与 8-06 版解读描述的源码差异；本地 `git fetch --unshallow` 被用户取消，未取上游 git diff / release notes）。
- 相比解读基线（2026-08-06 前后主分支）的主要变更：

**新增能力 / 能力点**
- moim 机制重构：`inject_moim`（就地注入现有 user 消息）→ `turn_context_message` / `turn_context_event`（按轮次组装成**独立 agent-only user 消息**持久化，带 `turn_context` 元数据）；重试经 `initial_messages` 快照保留已发送前缀；`system_prompt_block()` 把 turn-context 说明注入系统提示；`turn_budget_line` 更名 `turn_budget_part`（输出 `<turn-budget>` 标签）。
- 新增 `goose-agent`（GDK Agent Loop，inference 操作编排并入）、`goose-context-management`（会话压缩独立 crate）、`goose-roaming`（iroh 底座 P2P 漫游）、`goose-download-manager`、`goose-test`/`goose-test-support` 五个 crate。
- SQLite schema 升到 **v16**：新增 `idx_messages_session_created` 复合索引（会话列表 / 消息分页查询优化），演进从"加列"转入"索引先行"。
- 工具对先发压缩加入**去重**（`seen_message_pairs` HashSet 合并同条消息内的 sibling tool ids）+ `protect_last_n` 参数。

**交互与接入变化**
- `crates/goose-sdk` 对外命名改为 **GDK（Goose Development Kit）**。
- ACP 服务端能力声明新增 `delete`（`SessionDeleteCapabilities`，与 `close` 并列）。
- ACP `spawn_acp_process` 启动前把 `config.command` 父目录 prepend 到 `PATH`（兼容 `/usr/bin/env node` 式 npm 适配器）。
- ACP `Provider::stream()` 新增 `apply_effort_if_changed` 同步 thinking-effort；`messages_to_prompt` 参数 `false → None`。
- MCP `child_process_client` 改用 `spawn_long_lived_mcp_subprocess` 拉起子进程、`tokio::spawn` 后台读满 stderr，失败时把 stderr 拼进错误。

**接口 / API 变更**
- `categorize_tools` 签名新增 `toolshim_tools` 参数（本地推理伪装工具与常规工具分开分类）。
- `dispatch_tool_call` 前置 **PreToolUse hook** 硬拦截（denial 直接返回 ErrorData）；改用 `ToolCallContext` 显式承载 session/working_dir/request_id。
- `ProviderConstructor` 闭包新增 `use_default_model: bool`，命中时走 `F::from_env_with_default_model`。
- **context limit 统一到 provider API**：`get_context_limit` 默认实现经 `ContextLimitResolver`（按 provider 名查能力表），`crates/goose/src/context_limit.rs` 提供统一入口（先看 `GOOSE_CONTEXT_LIMIT` 覆盖、再落 provider API）。
- `handle_retry_logic` 签名去掉 `final_output_tool` 参数，改由 agent.rs 薄包装在 `Retried` 时清空（规避 `tokio::sync::Mutex` 不可重入）。
- `tool_stream` 从 `agent.rs` 移入 `tool_execution.rs`，产出 `ToolStreamItem`（ActionRequired / Message / Result）枚举。
- `execute_shell_command`：stdout 改 `/dev/null`，stderr 管道后 `capture_tail` 只留尾部 8KB（`MAX_COMMAND_STDERR_BYTES`）。

**行为 / 语义变化**
- 空轮处理：回合终结段 **recipe retry 优先接管**（跑 success-checks / on_failure / max_retries），仅 `Retried::Skipped` 且本轮为空时才走 `MAX_EMPTY_TURN_RETRIES` 有界空轮回退。
- `ProviderError` 处理：`ContextLengthExceeded` 压缩成功后 `yield HistoryReplaced` + `break` 出消化循环再重启轮次；新增 `Authentication` 变体分支；`CreditsExhausted` 改用 `SystemNotificationType::CreditsExhausted`（带 top-up 数据）；match 模式改用 `Err(ref provider_err @ …)` 绑定。
- `reply` 事件流新增 `.instrument(reply_span)` 保持 tracing span 上下文。

**Schema 与数据演进**
- 无新表；`messages` 新增复合索引（v16）。

**移除与废弃**
- `inject_moim`、`turn_budget_line`、`fix_conversation`（moim 内部）等旧符号随重构移除。
- `tool_stream` 在 `agent.rs` 中的定义移除（移至 `tool_execution.rs`）。
