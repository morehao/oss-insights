# 变更记录

> 本解读对 Codeg 的跨版本变更汇总；按版本倒序。描述清楚相关变动即可：若上游有官方 changelog，参照它并结合代码变动生成；没有则基于代码变动（git diff / release notes / 代码快照比对）生成。

## [0.30.0]

更新日期：2026-09-02。依据来源：**基于代码变动**（git diff `v0.28.2..v0.30.0`，无官方 changelog；上游 README 两版本哈希相同）。

### 新增能力

- **Conversation Canvas（会话画布）**：全新工作形态，把 folder/agent/会话/自定义区域/便签钉成卡片铺在无限画布上（34 文件 + 2 迁移 `canvas_node`/`canvas_node_group_grid`）；region 网格可钉列/行轴、整卡步进 resize；卡片可拖入会话展开成工作会话；revision 全序广播 + gap 回退快照，跨桌面/Web/远程同步；新增 8 个 `canvas_*` 命令与 `canvas://changed` 事件（见 [canvas.md](canvas.md)）
- **resume_delegation**：codeg-mcp 第 7 个工具，恢复被中断（取消/崩溃）的委托任务，重开子连接并沿用原 `delegation_call_id`；严格「延续不重跑」（不接受任务文本，`reason` 限长）
- **@-mention 路由帧**：`acp/agent_mentions.rs` 在每个 prompt 入口注入内部路由帧（RS 分隔符包裹、version 3），强制 `@agent` 委托走 `delegate_to_agent`、抑制 agent 原生 sub-agent；parser（codex/claude）负责从转录剥离该帧
- **sidebar folder groups**：folder 可归入用户命名分组（不嵌套、仅 regular folder），与散 folder 共享 `sort_order` 交错排列；canvas 新增 `kind=group` 区域；迁移 `folder_group`
- **work_task auto-compact**：resume 会话在下一轮 prompt 前按 `auto_compact_percent` 阈值自动发 compact 命令并等其落地（`work_task/compact.rs` 三级命令解析：显式覆盖 > 会话广告 > builtin default）
- **abandon 可带走 worktree**：放弃任务时可选保留 worktree，并提供事后删除入口
- **worktree 删除保护**：删除 worktree 前检查未提交工作，拒绝「带走未提交文件」（`CleanupBlocked` 双态区分「尝试失败」与「保护拒绝」）
- **Forge 增强**：自托管 GitLab 探测（`/api/v4/version` 而非靠主机名猜）、`resolve_forge_auth`/`host_profile` 单一派生规则、issue/PR 评论线程虚拟化滚动、新增 comment/issue/state/merge 命令、去 Beta 标记

### 交互与接入变化

- @-mentioned agent 路由帧取代「父 agent 自发调 delegate_to_agent」的隐含假设；子 agent 需配合路由帧指示
- cursor 参数化 model picker + thinking/Fast 旋钮修复；claude/codex 移除 Reasoning Effort 控件
- chat 端：assistant 回复按 round 折叠（持久 header）、折叠 completed turn progress
- 5 个 pinned agent CLI 滚动更新（codex/codex-acp/claude/gemini/opencode/qoder/antigravity/grok 等，见 `feat(acp): move five pinned agent CLIs`）

### Schema 与数据演进

- 迁移数 42 → 45（新增 `folder_group`、`canvas_node`、`canvas_node_group_grid`）
- service 20 → 22（新增 `canvas_service`、`folder_group_service`）
- `work_task_settings` 增 `auto_compact_percent`（默认 0）/ `compact_command` 字段
- `SessionState` 增 backend-only `delegation_enabled` 字段（是否注入路由帧的唯一 gate）

## [0.29.0]

更新日期：2026-09-02。依据来源：**基于代码变动**（git diff `v0.28.2..v0.29.0`，无官方 changelog）。

### 新增能力

- **Antigravity 无头登录**：`acp/antigravity_login.rs`（+1,578）支持 Linux 无桌面服务器上的 Google OAuth 无头登录（打印 URL + 回贴重定向），配套设置面板 `antigravity-config-panel.tsx`
- **Open in VS Code**：`commands/open_in.rs`，桌面/Server 形态把文件/目录交给 VS Code CLI 打开
- **归档会话 load-failure banner**：归档会话以已知的 load-failure banner 呈现，恢复命令可复制（含第三类 `session_load_failed` code）
- **sidebar 增强**：会话行旁浮出 folder/path/branch、显示会话 model（含 app 自启会话）、hover 气泡裁剪

### 交互与接入变化

- **Chat 会话折叠与快捷键**：assistant 回复按 round 折叠（持久 header）、completed turn progress 可折叠；Ctrl/Cmd+Shift+T 重开最后关闭的 tab；Ctrl/Cmd+1..9 跳 tab
- 6 个 pinned agent CLI 滚动更新（codex-acp 1.0.5→1.0.13 关键行为 pin，见 `feat(acp): move six pinned agent CLIs`）
