---
type: 项目解读 · 关注面深读
title: Codeg 前端工作台深度解读
description: Codeg 前端（Next.js/React 19）如何承载实时 agent UI：zustand 窄 selector store、reducer 事件模型、Split View 布局树、Conversation Canvas 页、流式消息分页与 seam hash 防串线
tags: [codeg, frontend, react, zustand, split-view, canvas]
---

# 前端工作台：把后端事件流投影成实时产品

> 本解读按 **ai-agent ⊗ framework** 类规格产出。这篇解读看用户实际面对的界面层——[acp-runtime.md](acp-runtime.md) 的事件流如何在 1088 个 ts/tsx 文件的前端落地成 Workspace / Split View / Conversation Canvas / Office 预览。**本版增量（v0.28.2 → v0.30.0）**：新增 Conversation Canvas 页面与闭卡栈/round 折叠等 store 面，ts/tsx 文件 1028→1088（[`CHANGELOG.md`](CHANGELOG.md)）。

## 根因：agent 会话是「永远在路上」的 UI

agent 回复按 token 流式到达、工具调用中途切换状态、会话可中断恢复、多个 agent 并行跑。这样的 UI 用传统「请求-响应 + 轮询」写不出来——必须**事件驱动**：后端推什么，前端投影什么。而桌面/多窗口/Web 三种形态共享同一套前端，事件模型必须与后端严格对齐。

## 事件模型：订阅-投影，而非拉取

前端不存真源。`src/lib/api.ts`（5.4K 行）是 Tauri IPC 的封装总入口，定义事件常量（如 `LOG_APPENDED_EVENT`、`FOLDER_OPEN_IN_WORKSPACE_EVENT`）并订阅后端事件流。会话级状态落在 `conversation-runtime-store`——它的 reducer 用**显式 Action 联合类型**驱动 [src/stores/conversation-runtime-store.ts:262]()：`FETCH_DETAIL_START/SUCCESS/ERROR`、`LOAD_OLDER_TURNS`、`COMPLETE_TURN`、`APPLY_BACKGROUND_ACTIVITY`、`RESOLVE_BACKGROUND_TASK`…… 每个动作都对应后端一个事件或一次命令响应，前端 reducer 与后端 `SessionState::apply_event` 保持**同一套单调合并规则**（这正是 [acp-runtime.md](acp-runtime.md) 里 `SessionFailure` 注释强调的前后端双实现）。

<!-- anchor: type Action = … -->

## Store 划分哲学：窄 selector，宽订阅再见

`tab-store` 的模块注释记录了最直接的演进决策[src/stores/tab-store.ts:48-64]()：**从单值 `TabContext`（每次 tab/状态更新都改变身份、rerender 全部约 20 个消费者，含无法 memo 屏蔽的 `ConversationTabView`）改为 Zustand 窄 selector 订阅**——消费者用 `useTabStore(selector)` 只订阅自己渲染的切片。这是桌面型实时 UI 的性能命门：一个高频事件（keep-alive、状态流转）如果触发整个会话面板重渲染，任何优化都救不回来。

## Split View：布局是一棵树，draft 是持久化的枝

README 的 Split View（拆右/拆下、拖拽 tab、布局记忆、draft 保留）落到 `tab-store` + `lib/tab-group-layout`：

- **布局模型**：`LayoutNode` / `SplitDirection` / `normalizeTree` / `resizeSplitAt` / `splitGroup` / `toggleOrientation`——分屏布局是显式的树结构，拖拽分割条即树的 resize，拆分组即树的 split [src/stores/tab-store.ts:15-26]()；
- **draft 保留**：`PersistedDraft` 与 `message-input-draft` 的存储 key 机制（`buildNewConversationDraftStorageKey`/`sweepOrphanDraftKeys`），重启后未发送的文本原样回到输入框；
- **虚拟运行时 id**：`TabItemInternal.runtimeConversationId` 是新会话的**虚拟负 id**，与持久化的 `conversationId` 解耦——draft 在绑定真实会话前先以虚拟身份活着 [src/stores/tab-store.ts:71-74]()；
- **agentTypeProvisional 纠正**：draft tab 的 agent 类型可能是 stale localStorage seed 或显示顺序 fallback 的**系统猜测**，标记 `agentTypeProvisional`，等 agent 列表刷新后用纠正效果替换——绝不把猜测当成事实渲染 [src/stores/tab-store.ts:80-93]()。

<!-- anchor: export interface TabItemInternal { … } -->

## 流式分页：tail 窗口 + seam hash 防串线

长会话不能一次渲染完。`conversation-runtime-store` 定义了分页语义[src/stores/conversation-runtime-store.ts:116-117]()：`TAIL_TURNS_DEFAULT = 120`（打开时渲染最近 120 轮）+ `OLDER_TURNS_PAGE_SIZE = 120`（向上翻页一次 120 轮）。

**seam hash 是防串线设计**：`PREPEND_OLDER_TURNS` 动作携带请求时的 `detail.prefix_hash`，reducer 合并前**重新校验当前 detail 的 hash**——分页窗口在途时可能已被替换（reset/refetch），不同 lineage 的页绝不能拼接 [src/stores/conversation-runtime-store.ts:290-301]()。这是把「网络竞态」显式建模进 reducer 的典型。

## 流式渲染管线

```mermaid
sequenceDiagram
    participant U as 用户
    participant CS as Composer
    participant API as lib/api
    participant RS as "conversation-<br>runtime-store"
    participant BE as Rust 后端（ACP）

    U->>CS: 输入消息
    CS->>API: 乐观插入用户 turn（本地）
    API->>BE: IPC 发送 prompt
    BE-->>API: 事件流（ContentDelta/ToolCall/…）
    API->>RS: dispatch(action)
    RS->>RS: reducer 应用（与后端同单调合并规则）
    RS->>CS: 订阅切片 → 增量渲染
    BE-->>API: TurnComplete
    API->>RS: "COMPLETE_TURN（liveMessage<br>sink 同步最终块）"
    RS->>CS: 收敛为持久化 turn

```

`registerLiveMessageSink` 让 `session.liveMessage` 随每批事件**同步写入**——面板不再各自订阅 `conn.liveMessage`，减少一层订阅竞态 [src/stores/conversation-runtime-store.ts:321-333]()。out-of-turn 活动（`APPLY_BACKGROUND_ACTIVITY`）与持久化详情之间用 [conversation-aggregation.md](conversation-aggregation.md) 讲的 `watermark` 交接，live overlay 不再与冷解析 turn 双渲染。

## Conversation Canvas：后端真源之外的第二种投影

v0.30.0 的前端多了整块 canvas 面（独立 route + ReactFlow 懒加载，协议细节见 [canvas.md](canvas.md)）。值得从 UI 工程角度看它的特征：

- **懒加载首访**：canvas-page 壳只在 `WORKBENCH_ROUTES` 注册页面，ReactFlow 与画布机械用 `next/dynamic` 延迟到首访才加载（画布视图读 `window` 做 viewport 数学，`ssr: false` 是显式声明）[src/components/canvas/canvas-page.tsx:14-21]()。

<!-- anchor: const CanvasView = dynamic(() => import("./canvas-view"), { -->

canvas-data 则先订阅 `canvas://changed` 再拉快照（tab-context 握手），杜绝「快照读到订阅生效之间」丢变更 [src/components/canvas/canvas-data.tsx:9-17]()。

<!-- anchor: * Wires the canvas store to the backend while the canvas page is mounted: … * so keeping the node set hot while it's closed buys nothing. -->

- **派生纯函数化**：`canvas-model.ts` 是「DB 节点 + live workspace state → ReactFlow graph」的纯派生层，交互态作为显式参数而非模块状态——整层可单测 [src/components/canvas/canvas-model.ts:11-15]()。

<!-- anchor: * Pure derivation layer for the conversation canvas: DB nodes + live workspace … * parameters, never as module state. -->

canvas-store 则把后端 revision 协议原样投影成前端收敛规则（事件流是唯一推进 `lastRevision` 的通道），`ReadonlyMap` 保证「只替换、不原地改」以便引用比较 [src/stores/canvas-store.ts:12-28]()。

<!-- anchor: * Canvas node cache + the revision protocol that keeps it convergent. … *    or are covered by the reconnect refetch. -->

同批 v0.29/0.30 前端新增 store/lib：`canvas-store`、`closed-tab-stack`（Ctrl+Shift+T 重开已关 tab）、`branch-tree`（worktree 分支分组）、`cursor-model-variants`（参数化 model picker）、`forge-list-prefs`/`forge-row-update`（issue/PR 评论虚拟化）、`image-tool-label`、`canvas-view-storage`（画布草稿/展开/视口持久化）。

## Office 实时预览与 Workspace

Office 文档（README 卖点）的前端侧：`html-preview-inline` / `@streamdown` 系列渲染器 + `html-to-image`，agent 每改一次 `.pptx/.docx/.xlsx`，`office_watch` 事件触发前端重新渲染同一文档——「无导出步骤、边改边看」。Workspace 的文件树/diff 面板/编辑器（Monaco）/终端（xterm）都围绕同一个「后端事件 → store 投影 → 增量渲染」管线，与聊天区共享 `app-workspace-store`。

## 全局关联

前端不是独立 UI，而是后端事件模型的**直接投影**：`conversation-runtime-store` 的 reducer 与后端 `SessionState::apply_event` 共享同一套单调合并语义（[acp-runtime.md](acp-runtime.md)）、`watermark` 交接来自会话聚合（[conversation-aggregation.md](conversation-aggregation.md)）、委托卡片消费 `DelegationStarted`/`DelegationCompleted`（[multi-agent-orchestration.md](multi-agent-orchestration.md)）。这正是「协议优先 + 会话真源」哲学的收尾验证：**前端没有任何自己的真源，它只是把后端唯一真源投影给用户看**——而 Split View 的树状布局与窄 selector store，是让这个投影在桌面级高频事件下依然流畅的工程答案。
