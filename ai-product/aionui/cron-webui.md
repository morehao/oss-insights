# Cron 定时任务 / WebUI 远程 / IM 渠道接入

> 仓库根：`/Users/morehao/Documents/study/ai/agent-product/AionUi`；引用为仓库相对路径。这三块共用同一架构基调：**调度、鉴权、渠道桥接的执行逻辑全在 Rust 后端 aioncore**，前端只负责「配置 UI + 状态订阅 + 生命周期编排」。凡断言跨到后端的标注【待主 agent 验证】。

## 一句话根因

把「无人值守的 Agent」变成现实：让 aioncore（Rust）作为唯一的调度/鉴权/IM 桥接中枢，前端只把用户的「自然语言任务 + 三种调度方式 + 会话绑定」翻译成 `/api/*` 契约，从而让 Agent 可在 Cron 触发、浏览器远程、IM 消息三种入口下共享同一套会话上下文与状态。

## Cron 定时任务（24/7 无人值守）

### 任务定义：三种调度 kind + 时区 + 会话绑定

调度模型是判别联合（discriminated union），三种 `kind` 对应三种调度方式：

```
[packages/desktop/src/common/adapter/ipcBridge.ts:1500-1503]() ICronSchedule 判别联合：at/every/cron(带时区)
<!-- snippet: export type ICronSchedule =
  | { kind: 'at'; atMs: number; description: string }
  | { kind: 'every'; everyMs: number; description: string }
  | { kind: 'cron'; expr: string; tz?: string; description: string }; -->
```

任务本体 `ICronJob` 把「会话绑定上下文」建模在 `metadata.conversation_id` 与 `target` 上（`ipcBridge.ts:1505-1534, 1511-1522`）：

- `execution_mode: 'existing'` → 复用绑定的会话，把 `target.payload.text` 作为消息注入，天然继承历史上下文；
- `execution_mode: 'new_conversation'` → 每次执行开新会话，但新会话挂 `cron_job_id`，前端据此把"同一次任务的历次运行"归拢到一个 job 下。

```
[packages/desktop/src/common/adapter/ipcBridge.ts:1511-1514]() target.payload + execution_mode：会话绑定如何建模
<!-- snippet:   target: {
    payload: { kind: 'message'; text: string };
    execution_mode?: 'existing' | 'new_conversation';
  }; -->
```

### 自然语言建任务 → 前端编译成 schedule

`CreateTaskDialog.tsx` 是前端侧的调度编排中心：用户选频率 + 填 prompt，前端合成 5 段 cron 表达式（`buildCustomCronExpression` 处理 `*/N`、`MON-FRI` 等）并锚定本地时区，然后 `POST /api/cron/jobs`：

```
[packages/desktop/src/renderer/pages/cron/ScheduledTasksPage/CreateTaskDialog.tsx:529-531]() 提交时补当前时区
<!-- snippet:       const scheduleExpr = scheduleInfo.expr;
      const scheduleDesc = scheduleInfo.description;
      const schedule = createCronSchedule(scheduleExpr, scheduleDesc); -->
```

新会话归拢到同一任务由 `cronUtils.resolveCronJobId` 实现，兼容 `cron_job_id`（snake）与 `cronJobId`（camel）两种旧数据（`cronUtils.ts:129-136`）。

### 时区自愈

因 `tz` 可选，旧任务可能缺时区。前端 bootstrap 时对所有 job 跑一次"缺时区就补当前时区"的幂等修复，用 `Map<job_id, Promise>` 去重防并发修补（`repairCronJobTimeZone.ts:30-39`）。

### 保持唤醒 + 错过触发恢复 —— 前端只补"resume 通知"

- **保持唤醒**：前端把 `system.keepAwake` 设置键双写本地 config + 下发给后端（`POST /api/settings/client`）。本地 Electron 的 `powerSaveBlocker.start('prevent-app-suspension')` 封装在 `common/platform/ElectronPlatformServices.ts`，但最终生效点是否纯依赖 Electron 需要后端确认【待主 agent 验证】。
- **错过触发**：前端主进程监听系统 resume，向后端补发一次内网通知，让后端补跑休眠期间错过的触发：

```
[packages/desktop/src/index.ts:306-312]() 系统 resume → 向后端补发 system-resume
<!-- snippet:   const onResume = () => {
    void fetch(`http://127.0.0.1:${backendPort}/api/cron/internal/system-resume`, {
      method: 'POST',
      headers: { 'x-aionui-internal': '1' },
    }).catch(...);
  };
  powerMonitor.on('resume', onResume); -->
```

> 这一机制意味着：**错过触发的"检测 + 补跑 + 标 missed"逻辑全在 aioncore**，前端侧只能看到 `last_status: 'missed'` 状态与红色徽标。

### "到点触发 agent 执行"——执行完全在后端

前端侧没有任何调度执行器。前端只做 CRUD + 状态订阅（`ipcBridge.cron` 全走 `/api/cron/*`，含 `listJobs/addJob/updateJob/removeJob/runNow/saveSkill/hasSkill`），to-point 触发、missed/queue 状态机、Skill 持久化全在后端。桌面侧数据库的 `cron_jobs` 建表（`migrations.ts:263`）是历史遗留——现代版本在 aioncore 侧建表，桌面这份仅剩 legacyHandoffContract 迁移字段。

## WebUI / 远程访问

### 拓扑：两层 HTTP（TCP 前置路由）

`@aionui/web-host` 的 `static-server.ts` 用 Node 原生 `http` + serve-handler 实现，并刻意用双层 TCP listener（外层 `net.Server` 首字节 peek 路由 + 内层 `http.Server` 非 upgrade 请求）：

- `allowRemote=true` 监听 `0.0.0.0`，否则 `127.0.0.1`。
- 外层 TCP 命中 `GET /ws` 或 `GET /api/stt/stream` 直接 raw TCP splice 到后端（WebSocket/STT 流直通，不经过前端反代）。
- 其余路由：`/api/*`、`/login`、`/logout` 反代给后端，否则当 SPA 静态服务 + `index.html` 回退。

```
[packages/web-host/src/static-server.ts:144-147]() 鉴权交给后端，前端只透传
<!-- snippet:       if (req.url.startsWith('/api/') || req.url.startsWith('/api?') || req.url === '/login' || req.url === '/logout') {
        forwardToBackend(req, res, opts.backendPort);
        return;
      } -->
```

### 鉴权：密码 login + QR 登录（契约层面）

- 用户名/密码/初始密码的种子与重置由后端 users 表承载，`webui.changePassword/resetPassword/generateQRToken` 全走 `/api/webui/*`（`ipcBridge.ts:1441-1448`）。
- QR 登录：前端 `generateQRToken` 拿 token + 过期时间，拼出 `{baseUrl}/qr-login?token={token}` 渲染二维码；token 5 分钟过期，前端 4 分钟自动刷新。
- 首次启用：探测 `/api/auth/status` 的 `needs_setup`，为真则调 `/api/webui/reset-password` 让后端生成随机密码，明文一次性展示（`webuiBridge.ts:52-77,86-101`）。
- 鉴权完全委托给 aioncore 的 `aionui-auth` crate（`static-server.ts:1-11` 注释明言）。

### agent-process-registry：进程登记，不是"远程控制"

易误读点：`packages/web-host/src/agent-process-registry.ts` 的 `RegisteredAgentProcess`（pid / process_group_id / conversation_id）**并不承载 WebUI 对 agent 的"远程控制"**。它的职责是**孤儿进程回收**（后端停时读 registry，SIGTERM→宽限 1s→SIGKILL）。真正"远程控制 Agent"走的是 WebUI 前端正常 `/api/*` 反代到 aioncore。这是一层安全边界——避免远程会话把进程管理能力暴露出浏览器。

### 部署形态

- 纯 CLI/Bun 模式：`scripts/webui.ts` 独立于 Electron，可 headless（`AIONUI_HOST=0.0.0.0`），独立数据目录防 symlink 冲突。
- Electron `--webui`：复用桌面 data-dir 使会话/定时任务"在哪都能看到"（`index.ts:865-917`），用 `useExistingBackend` 复用已拉起 aioncore，避免第二后端竞争 SQLite。

## 聊天平台接入：一套通用"渠道 = 插件"契约

前端侧本质是一套通用配置面板，五种 IM（Telegram/Lark/DingTalk/WeChat/WeCom）复用同一套配对/授权/桥接 UI 与后端起停契约（`channel` IPC，全走 `/api/channel/*`，`ipcBridge.ts:1995-2041`）。

- **插件生命周期**：`getPluginStatus/enablePlugin/disablePlugin/testPlugin`。`testPlugin` 传入 `{plugin_id, token, extra_config?}`（DingTalk/Lark 需 app_id/app_secret），返回 `{success, bot_username?, error?}`。
- **配对授权**：用户在 IM 里发消息触发配对请求 → WebSocket `channel.pairing-requested` 推到前端 → `approvePairing/rejectPairing` → `channel.user-authorized` 热更新授权列表。
- **事件驱动**：`pluginStatusChanged` 让渠道卡片实时反映 connected/botUsername。

以 Telegram 为例：填 Bot Token → `testPlugin` → 成功即 `enablePlugin` 自动启用 → 用户在 `/start` 触发配对 → `pairingRequested` → `approvePairing` → 授权后 `revokeUser` 可收权（`TelegramConfigForm.tsx:199-266,280-288`）。Token 一有授权用户即锁定，避免中途改 token 造成安全裂缝。

桥接执行（长轮询 Bot API / 处理 IM webhook → 注入 agent 会话 / streaming / fallback）全在 aioncore 渠道插件端，前端只看得到 `channel.*` 契约【待主 agent 验证后端实现】。

## 关键设计权衡（Why）：24/7 无人值守的可靠性

| 难点 | 系统选择 |
|------|----------|
| 调度必须独立于 UI 存活 | 调度器放 Rust 后端，前端崩溃/关窗不影响 |
| 时区不一致导致误触发 | cron 表达式自带 IANA tz + 旧数据自愈 |
| 系统休眠错过触发 | 主进程监听 `powerMonitor.resume` 补发 `system-resume` |
| 保持唤醒防休眠 | 前端把 `keepAwake` 下发后端，原生 powerSaveBlocker 封装在 platform 层 |
| 新/单会话两种模式共享上下文 | `conversation_id` + `execution_mode` 显式建模 |
| WebUI/IM 暴露公网风险 | 鉴权统一走后端 aionui-auth（密码/QR/JWT）；agent 进程管理不暴露给远程 |
| 独立宿主可一键 headless | `@aionui/web-host` 零 Electron 依赖，被 CLI/Electron/系统服务复用 |

最深刻的一条权衡是**「可靠性在哪里落」**：Cron 的到点触发、错过补跑、missed 状态机，WebUI 鉴权，IM 配对授权，全部从渲染/主进程剥离、下沉到常驻 aioncore。只要 aioncore 活着，任务就能跑，与 Electron 窗口是否打开、浏览器连接是否断开无关。

## 亮点与问题

- **亮点**：`execute` 全在后端让前端变"薄 UI"；resume 补发通知是省事又有效的错过恢复设计；channel 通用插件契约让新增 IM 只需填 token 形态。
- **问题**：跨进程归因难（missed 细节后端不可见）；keepAwake 最终生效点跨仓库；WebUI 对公网时攻击面显著增大。

## 与其他模块的协同

- `cron_jobs`/`assistant_plugins`/`assistant_pairing_codes` 表在数据模型中定义（[`data-model.md`](data-model.md)）。
- WebUI/IM 的 agent 执行复用到 ACP 会话模型（[`multi-agent-acp.md`](multi-agent-acp.md)）。
- 所有 `/api/*` 依赖引擎启动链路（[`architecture.md`](architecture.md)）。

> 【待主 agent 验证】Cron 到点触发/missed/queue 状态机、keepAwake 生效点、QR/JWT 轮转、IM 渠道执行——均在 aioncore（AionCore）Rust 侧。

## 主要源码文件

- `renderer/pages/cron/cronUtils.ts` / `useCronJobs.ts` / `repairCronJobTimeZone.ts`
- `renderer/pages/cron/ScheduledTasksPage/{index,CreateTaskDialog}.tsx`
- `packages/web-host/src/static-server.ts` / `agent-process-registry.ts` / `index.ts`
- `process/bridge/webuiBridge.ts` / `process/utils/webuiConfig.ts`
- `renderer/.../contents/WebuiModalContent.tsx`
- `renderer/.../contents/channels/TelegramConfigForm.tsx`
- `common/config/configKeys.ts`（keepAwake 键）
