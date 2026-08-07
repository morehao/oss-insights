# 多 Agent 集成与 ACP + Team Mode

> 仓库根：`/Users/morehao/Documents/study/ai/agent-product/AionUi`；引用为仓库相对路径。真正的 ACP server / Team MCP server 实现在独立仓库 AionCore（Rust 后端），本仓库只覆盖 Electron 前端侧的 ACP 会话模型、事件映射、Team Mode 状态机。凡断言跨到后端的一律标注【待主 agent 验证】。

## 一句话定位

前端把「agent 运行时」收敛成唯一判别字段 `conversation.type`（`acp` / `aionrs` 两个活的，其余 deprecated），ACP 成为接入外部 CLI agent 的统一会话协议；Team Mode 在 ACP 会话之上再叠一层 Leader ↔ Teammate 编排状态机（run / slot_work / mailbox / task board），全部经 `/api/teams/*` + WebSocket 事件驱动。

## 在项目中的角色

- 去掉它：AionUi 回到"只支持内置 aionrs 一个引擎"的状态，README 宣称的"几十种 CLI agent 一个统一界面"消失，Team Mode 多 Agent 协作（核心卖点之二）完全丢失。
- 它是整个渲染层最重的关注面，直接驱动 `renderer/pages/conversation/` 与 `renderer/pages/team/` 两套页面。

## 运行时收敛到 ACP：几十种 CLI agent 统一成 `acp` 类型

唯一生效的运行时判别是 `acp`（外部 CLI agent 统一外壳）与 `aionrs`（内置 Rust 引擎），其余（openclaw-gateway / nanobot / remote / gemini）整体标 deprecated：

```
[packages/desktop/src/renderer/utils/model/agentTypeSupportPolicy.ts:7-9]() 运行时收敛到 ACP
<!-- snippet: export function resolveSupportedConversationType(backend?: string | null): 'acp' | 'aionrs' {
  return backend === 'aionrs' ? 'aionrs' : 'acp';
} -->
```

`DEPRECATED_RUNTIME_AGENT_TYPES`（`agentTypeSupportPolicy.ts:1`）显式列出被淘汰的运行时类型，`acp` 会话内再以 `conversation.extra.backend` 细分真实 CLI（claude/codex/…）。前端不再需要为几十个 agent 各写一套会话逻辑——这就是"一个统一界面串起几十种 CLI"的最小实现路径。

## ACP 能力协商：统一抽象成本的答案

ACP 的 `initialize` 响应（JSON-RPC result）被完整建模为 `AcpInitializeResult`（`packages/desktop/src/common/types/platform/acpTypes.ts:91-100`），含 `protocolVersion`、`capabilities`（`AcpAgentCapabilities`：prompt / mcp / session 三族）、`agentInfo`、`auth_methods`。UI 据此决定渲染哪些交互（图片/音频输入、MCP 开关、fork/resume 等）：

```
[packages/desktop/src/common/types/platform/acpTypes.ts:95-100]() ACP 初始化结果含能力协商，UI 据此差异渲染
<!-- snippet: export type AcpInitializeResult = {
  protocolVersion: number;
  capabilities: AcpAgentCapabilities;
  agentInfo: AcpAgentInfo | null;
  auth_methods: AcpAuthMethod[];
}; -->
```

`AcpMcpCapabilities`（`acpTypes.ts:39-43`）枚举 `stdio/http/sse` 三种 transport，`sessionCapabilities` 含 `fork`/`resume`/`list`/`close`（key 存在即支持）。Team Mode 是否可用也由能力决定——`mcpCapabilities.stdio` 是后端算 `team_capable` 的依据【待主 agent 验证后端 compute 实际逻辑】。

## ACP 会话生命周期

`useAcpMessage.ts` 是全 ACP 会话的"状态机 + 事件映射中枢"，订阅 `ipcBridge.acpConversation.responseStream`（WS `message.stream`），把后端归一化后的 `IResponseMessage` 逐个 case 落成 UI 状态：

```
[packages/desktop/src/renderer/pages/conversation/platforms/acp/useAcpMessage.ts:114-116]() ACP 会话状态竖井：connecting→connected→authenticated→session_active→disconnected→error
<!-- snippet: const [acpStatus, setAcpStatus] = useState<
  'connecting' | 'connected' | 'authenticated' | 'session_active' | 'disconnected' | 'error' | null
>(null); -->
```

`agent_status` 事件把认证完成/断线/错误的流转落到状态（`useAcpMessage.ts:415-429`）。turn 生命周期由 `start`/`finish`/`error` 驱动。会话切换用 `isConversationProcessing` 水合逻辑重放 running 状态。

ACP 会话元数据持久化在 `acp_session` 表（`migrations.ts:1194-1209`，存 `session_id`/`session_status`/`session_config` + 局部索引支撑恢复）【待主 agent 验证：与后端 session 的同步/恢复策略】。

## 事件归一化：一个 `switch(message.type)` 吃下全部 ACP 事件

ACP 的 tool_call / permission / plan / usage 被后端转成 `IResponseMessage.type`，前端一个 switch 分发：

```
[packages/desktop/src/renderer/pages/conversation/platforms/acp/useAcpMessage.ts:276-277]() 事件归一化：switch(message.type) 一个函数吃下全部 ACP 事件
<!-- snippet:   const transformedMessage = transformMessage(message);
  switch (message.type) { -->
```

type 体系包括：`thought`/`thinking`/`start`/`request_trace`/`acp_context_usage`/`acp_model_info`/`acp_config_option`/`codex_model_info`/`available_commands`/`agent_status`/`user_content`/`teammate_message`/`acp_permission`/`acp_terminal_output`。后端双写兼容各家后门字段（`rawOutput`/`raw_output`、`acp_model_info`/`codex_model_info` 并存），前端用安全默认值兜底——标准化协议 + 各家后门字段兼顾：

```
[packages/desktop/src/common/chat/acpToolCallOutput.ts:58-62]() ACP 双写兼容各家后门字段
<!-- snippet: export const sanitizeAcpToolUpdate = (update: ToolCallUpdate['update']): ToolCallUpdate['update'] => ({
  ...update,
  rawOutput: sanitizeAcpRawOutput(update.rawOutput),
  raw_output: sanitizeAcpRawOutput(update.raw_output),
}); -->
```

工具调用呈现为 diff / shell / 图片卡片，`kind` ∈ read/edit/execute（`MessageAcpToolCall.tsx:89-100`）。权限请求建模成统一卡片 `MessageAcpPermission.tsx`。

## Team Mode：每成员 = 独立 ACP 会话，Leader 通过 Team MCP Server 编排

Team 数据结构是 `slot_id → conversation_id` 的映射，每个 assistant 携带 `role: 'leader'|'teammate'` 与自己的会话：

```
[packages/desktop/src/common/types/team/teamTypes.ts:20-32]() TeamAssistant：slot_id + conversation_id + role + status
<!-- snippet: export type TeamAssistant = {
  slot_id: string;
  conversation_id: string;
  role: TeammateRole;
  assistant_backend: string;
  icon?: string;
  assistant_name: string;
  status: TeammateStatus;
  cli_path?: string;
  assistant_id?: string;
  model?: string;
  pending_confirmations?: number;
}; -->
```

### 状态机

前端把每个 slot 的多轮并行工作建模成 `slot_work`（state ∈ idle/queued/starting/running/paused/blocked），配合 team run（`runAccepted→runStarted→runUpdated→runCompleted/Cancelled/Failed`）驱动整体状态机：

```
[packages/desktop/src/common/types/team/teamTypes.ts:66-68]() Team run 状态机 + slot_work 状态机
<!-- snippet: export type TeamRunStatus = 'accepted' | 'running' | 'cancelling' | 'completed' | 'cancelled' | 'failed';
export type TeamSlotWorkState = 'idle' | 'queued' | 'starting' | 'running' | 'paused' | 'blocked';
export type TeamSlotBlockedReason = 'runtime_starting' | 'runtime_failed' | 'removing' | 'session_stopped'; -->
```

### 异步 mailbox 与 task board

Team 消息异步投递模型：`ITeamMailboxMessage` 带 `from_agent_id`/`to_agent_id`/`summary`/`read`，广播用保留目标 `'*'`（`teamTypes.ts:245-256`）。活动视图把 mailbox 与 task board 合并成统一时间轴 feed（`ITeamActivityItem` = message | task，`teamTypes.ts:273-275`），后端合并双流、前端单 keyset 游标分页 + WS 增量 upsert（`useTeamActivityFeed.ts`）。

任务板任务项带 owner + 依赖环：

```
[packages/desktop/src/common/types/team/teamTypes.ts:259-270]() task board 任务项：subject + owner + blocked_by + blocks
<!-- snippet: export type ITeamTaskItem = {
  id: string;
  team_id: string;
  subject: string;
  description?: string;
  status: string;
  owner?: string;
  blocked_by: string[];
  blocks: string[];
  created_at: number;
  updated_at: number;
}; -->
```

### Leader 任务拆解 — 前端只投不拆

前端把用户输入投给 Leader 会话（`team.sendMessage`），其余成员走 `team.sendMessageToAgent`；Leader 实际拆任务、建 task、派发给 teammate 的推理与 MCP 调用在 AionCore 后端内部，前端不参与。发送入口是 `ipcBridge.team.sendMessage` → `POST /api/teams/{id}/messages`（`ipcBridge.ts:2147-2153`）：

```
[packages/desktop/src/renderer/pages/team/components/TeamChatView.tsx:153-164]() 前端只投给 Leader，不拆任务
<!-- snippet: const teamSendMessage = useCallback<TeamSendOverride>(
  async ({ input, files }) => {
    if (!team_id) throw new Error('Missing team id for team send');
    if (isLeader) {
      const ack = await ipcBridge.team.sendMessage.invoke({ team_id, input, files });
      onTeamRunAck?.(ack);
      return;
    }
    if (!slot_id) throw new Error('Missing slot id for team agent send');
    const ack = await ipcBridge.team.sendMessageToAgent.invoke({ team_id, slot_id, input, files });
    onTeamRunAck?.(ack);
  }, [isLeader, onTeamRunAck, slot_id, team_id]); -->
```

### 并行视图与权限分隔

每个成员一列（`flex: 1 1 400px`，leader 列带 crown），共享一个 run 状态机（`TeamPage.tsx:576-616`）。权限分隔以「独立 conversation_id + 独立运行时」隔离成员，`useTeamPermission` 用 null 检查把 team 逻辑与单 agent 模式隔开（`TeamPermissionContext.tsx:85-91`）。会话就绪用幂等 `warmupSession`（`TeamPermissionContext.tsx:43-56`），WS 事件为权威 + 手动 reconcile 为兜底。

## 外部 agent vs 内置 agent：架构同构，差异在会话启动

外部：`acp` 会话 + ACP 能力协商 + `backend` 细分；内置 aionrs：`aionrs` 会话 + 独立模型选择。二者共享 TeamChatView 的 send/run/status 外壳（`TeamChatView.tsx:228-265`）。演进趋势：`DEPRECATED_RUNTIME_AGENT_TYPES` + "Gemini → ACP consolidation" 注释（`teamCreateModelResolver.ts:96-101`），说明内置以外正逐步迁移到 ACP 一个标准。

## 关键设计决策

| 决策 | 权衡 |
|------|------|
| 用 ACP 标准协议 | 统一会话模型让 UI 只需写一套 case 分发；代价是各家后门字段仍需兼容垫片，无法 100% 消灭"某家专属" |
| 能力协商驱动 UI | 差异变成数据而非代码；代价是 `config_option` 协议在异步 CLI 下可能出现 `config_not_observed` |
| 状态一致性：WS 事件 + 手动 reconcile | 不依赖最终一致的后端快照，但增加缓冲区逻辑 |

## 亮点与问题

- **亮点**：运行时收敛到 `acp`/`aionrs` 是最简洁的入口；`switch(message.type)` 事件归一化用一个函数覆盖全部 ACP 事件；Team Mode 的 slot_work 状态机让并行编排可视化。
- **问题**：大量编排逻辑在后端，前端边界模糊；`acp_session` 表的持久化/恢复策略需要跨仓库验证才能确定完整性。

## 与其他模块的协同

- ACP 会话的 MCP 能力声明（`acpTypes.ts:39-43`）与 MCP 统一管理对接（[`mcp.md`](mcp.md)）。
- `acp_session` 表在数据模型中定义（[`data-model.md`](data-model.md)）。
- 所有 `/api/*` 调用依赖引擎启动后的端口（[`architecture.md`](architecture.md)）。

> 【待主 agent 验证】AionCore 中 ACP server（stdio transport）与 Team MCP server 的真实实现；Leader「拆任务 + 分派」的 agent 侧指令；`acp_session` 与后端 session 的同步/恢复策略。

## 主要源码文件

- `common/types/platform/acpTypes.ts` — ACP 协议类型
- `packages/desktop/src/common/types/team/teamTypes.ts` — Team Mode 类型
- `common/chat/acpToolCallOutput.ts` — ACP 工具调用处理
- `packages/desktop/src/renderer/pages/conversation/platforms/acp/useAcpMessage.ts` — ACP 会话状态机 + 事件映射
- `renderer/.../team/TeamPage.tsx` + `TeamChatView.tsx` — Team Mode UI
- `renderer/.../team/hooks/useTeamRunView.ts` — Team run 状态机
- `renderer/.../team/hooks/TeamPermissionContext.tsx` — 权限与 warmup
- `renderer/.../team/activity/useTeamActivityFeed.ts` — 活动流订阅
- `renderer/utils/model/agentTypeSupportPolicy.ts` — 运行时收敛