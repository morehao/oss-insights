# 变更记录

> 本解读对 AionUi 的跨版本变更汇总；按版本倒序。描述清楚相关变动即可：若上游有官方 changelog（仓库根 `CHANGELOG.md`），参照它并结合代码变动生成；没有则基于代码变动（git diff / release notes / 代码快照比对）生成。本文件是版本说明而非技术解构，不参与真源引用校验。

## [2.2.1]（2026-09-01 发布）

> 本次增量（2026-09）：依据上游官方 CHANGELOG（v2.1.61…v2.2.1）与 `git diff v2.1.61..HEAD`，并跑 `verify-references.mjs --diff` 全量引用预检（STABLE=283 / STALE=25 / UNRESOLVED=0，STALE 全部本次校正，无 UNRESOLVED）。

- **引用与现状校准（全 corpus）**：上游 #4198 移除 `ipcBridge.conversation.confirmation.add/update` 两个从未被后端触发的死 emitter，并把 `useTeamPendingPermissions` / `useSiderTeamBadges` 里的 live +1 订阅一并删除 → 所有 ipcBridge 引用整体 −2 行、team pending 两 hook 引用 −4 行，涉及 acp / automation / mcp-skills / remote-access / team-mode 五篇共 25 处 STALE 引用已按真实行号修正；`architecture.md` 的 ipcBridge 行数表述 2508 → 2506。
- **交互语义变更**：团队待确认徽标不再"订阅 add/remove 实时增减"——改为「`confirmation.list` / 团队摘要 seed + 仅 `confirmation.remove` 递减」，新增确认经 `message.stream` 的 permission / acp_permission / ask 帧到达（上游 #4198 顺手把会话侧边栏做成"等待用户"态并通知）。`team-mode.md`「权限对话框与侧边栏徽标」一节改写。
- **版本表述**：壳 v2.1.61 → v2.2.1、引擎 aioncore v0.1.72 → v0.2.1（README / architecture / remote-access 头部同步）。
- **上游同窗口其他变更（登记，未触及既有章节论断）**：#4195 turn 完成通知带会话名、#4198 会话侧边栏等待态——会话列表/通知 UI 不在既有解读章节覆盖内，仅在此汇总；#4175（WebUI 401 静默续期 + WS 1008 续期后重连）与 #4156（bridge WS 重连退避修复）在解读基线（2026-08-28 main 快照）已覆盖，正文无改动。
- **引擎侧（aioncore 外部仓库，本仓库不可验证）**：v0.2.x 含 BREAKING「skills 改经 AionUi-owned view 交付而非 workspace」（#938）与双 token 续期 singleflight（#926）。涉及文档已标注"依赖 aioncore 外部实现"边界，留待对照 AionCore 核实，本次不改正文。

## [2.1.61]（2026-08-25 发布）

> 解读 corpus 基线版本：README 与七个子文档于 2026-08-28 前后落成，对应当时代码快照（main，已含其后并入 v2.2.1 的 #4175/#4156），版本标签按当时最新 release v2.1.61 标注。
