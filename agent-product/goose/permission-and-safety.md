# 权限、安全与审批

> 覆盖 goose 的工具调用权限治理与安全防护子系统。

## 根因——为什么需要这个模块

goose 是跨桌面 / CLI / API 三形态的通用 Agent，主循环里让模型调用任意本地工具的权限与能力越界，是本模块要拦截的根本问题。

一句话锚点：**工具调用是 Agent 伸向真实世界的"手"，权限模块决定这只手能不能动，安全模块决定这只手会不会被恶意输入劫持。**

两条独立但最终汇合的安全主轴：

- **权限维度（信任谁）**：用户想给 agent 多大自由度。`Auto` 全放行、`Approve` 每次询问、`SmartApprove` 只对敏感工具询问、`Chat` 完全不执行。模式由 `goose_mode.rs` 定义。
- **安全维度（输入可信吗）**：工具的**参数和调用上下文**都是不可信数据，可能夹带 prompt 注入 / 破坏性命令。

两者在主循环里被统一的 `ToolInspectionManager` 编排成一条决策链，最终汇合成三类结果：`Allow`（执行）、`NeedsApproval`（唤起人类审批）、`Deny`（直接拒绝）。

## 架构与主循环链路

### 线程编排总览

```mermaid
graph TB
    LLM["模型返回 ToolRequest 列表"]
    INS["ToolInspectionManager 逐个跑 inspector"]

    SEC["SecurityInspector<br>pattern+ML 判 shell"]
    EGR["EgressInspector<br>出站流量检查"]
    ADV["AdversaryInspector<br>LLM 规则审查"]
    PERM["PermissionInspector<br>annotation+store+LLM"]
    REP["RepetitionInspector"]

    PROC["process_inspection_results<br>permission 为 baseline<br>其他 inspector 作 override"]
    PCR["PermissionCheckResult<br>approved/needs_approval/denied"]

    LLM --> INS
    INS --> SEC
    SEC --> EGR
    EGR --> ADV
    ADV --> PERM
    PERM --> REP
    REP --> PROC
    PROC --> PCR
    PCR --> DISP["dispatch_tool_call 直接执行"]
    PCR --> DENY["handle_denied_tools 回 DECLINED_RESPONSE"]
    PCR --> APPR["handle_approval_tool_requests 审批握手"]
```

说明：四个 inspector 的**加入顺序即优先级**，security（最高）→ egress → adversary → permission（中高）→ repetition（最低），见 `agents/agent.rs:715-742`。

### 主循环如何把决策链串起来

eval 拿到模型产出的 `remaining_requests` 后，先调 `inspect_tools` 跑全部 inspector，得到 `inspection_results`（`agents/agent.rs:2377-2386`）。随后 `process_inspection_results_with_permission_inspector` 把这些结果**加工成三类** `PermissionCheckResult`（`agents/agent.rs:2388-2401`），再分别交给三条执行路径（`agents/agent.rs:2413-2433`）。

加工逻辑本身在 `process_inspection_results`（`permission/permission_inspector.rs:73-131`）：先取 `inspector_name == "permission"` 的结果作 baseline，然后 `apply_inspection_results_to_permissions`（`tool_inspection.rs:170-261`）让其它 inspector 的 `Deny`/`RequireApproval` 覆盖 baseline 的 `Allow`。核心语义：

- 任何 inspector 拒绝 → 移到 `denied`
- 任何 inspector 要求审批 → 移到 `needs_approval`（仅当不属于 denied）
- 单个 inspector 的 `Allow` **不会越权**覆盖别人的 deny/approval（`tool_inspection.rs:253-256`，安全最大化：宁严勿宽）

> `[crates/goose/src/tool_inspection.rs:253-256]()` — "Allow" 分支不覆盖他人决定
> <!-- snippet-base64:ICAgICAgICAgICAgSW5zcGVjdGlvbkFjdGlvbjo6QWxsb3cgPT4gewogICAgICAgICAgICAgICAgLy8gVGhpcyBpbnNwZWN0b3IgYWxsb3dzIGl0LCBidXQgZG9uJ3Qgb3ZlcnJpZGUgb3RoZXIgaW5zcGVjdG9ycycgZGVjaXNpb25zCiAgICAgICAgICAgICAgICAvLyBJZiBpdCdzIGFscmVhZHkgZGVuaWVkIG9yIG5lZWRzIGFwcHJvdmFsLCBsZWF2ZSBpdCB0aGF0IHdheQogICAgICAgICAgICB9 -->
> -->

### 决策链时序图

```mermaid
sequenceDiagram
    participant LLM as 模型主循环
    participant MGR as ToolInspectionManager
    participant PERM as PermissionInspector
    participant SEC as SecurityInspector
    participant RTR as ToolConfirmationRouter
    participant EXEC as 工具执行

    LLM->>MGR: remaining_requests
    Note over MGR: 按注册顺序逐个 inspect（security→egress→adversary→permission→repetition）
    MGR->>SEC: 扫描 shell 命令 / 对话上下文
    SEC-->>MGR: 恶意→RequireApproval(带🔒告警)
    MGR->>PERM: 打分（user store → annotation → LLM 判只读 → 兜底询问）
    PERM-->>MGR: Allow/Deny/RequireApproval
    MGR->>MGR: process_inspection_results 汇合三类

    alt approved
        MGR->>EXEC: dispatch_tool_call 直接执行
    else needs_approval
        MGR->>RTR: register(request.id) → oneshot rx
        MGR-->>前端: action_required 消息
        前端-->>RTR: deliver(confirmation)
        alt AllowOnce/AlwaysAllow
            RTR-->>MGR: 放行 → dispatch_tool_call
        else AlwaysDeny
            MGR->>MGR: 回 DECLINED_RESPONSE + 写 NeverAllow
        end
    else denied
        MGR->>MGR: 回 DECLINED_RESPONSE
    end
```

---

## 关键设计·深挖

### 决策一：PermissionInspector 的分级裁决优先级

`permission/permission_inspector.rs:159-195` 是权限决策的"主干"，它对 `Approve`/`SmartApprove` 两种模式按固定顺序做五级判断：

1. **用户显式配置最优先**（`get_user_permission`）：`AlwaysAllow`→Allow、`NeverAllow`→Deny、`AskBefore`→问（`permission_inspector.rs:164-171`）。用户写的规则压过一切智能推断。
2. **SmartApprove 下的只读注解**：工具带 `read_only_hint == true` 注解则直接 Allow（`permission_inspector.rs:173-176`）。**注意**这个设计只在 SmartApprove 下生效，Approve 模式下再只读也要问（测试 `approve_ignores_annotation` 印证，`permission_inspector.rs:326`）。
3. **扩展管理工具特判**：`manage_extensions` 无脑 RequireApproval 并带安全告警（`permission_inspector.rs:178-181`）——危险能力不信任任何缓存。
4. **SmartApprove 且未缓存 / 只缓存过 Allow**：交给 LLM 判只读（`permission_inspector.rs:183-190`）。这是 smart_approve 的核心：把"这个工具这次调用是否安全"的判断授权给模型。
5. **兜底**：一切未知工具 RequireApproval（`permission_inspector.rs:192-193`）——默认拒绝，宁可问不可放。

这五级的顺序本身就体现权衡：**用户的显式意愿 > 静态注解 > 模型动态推理 > 默认拒绝**。硬规则决定会不会问，软规则（LLM）决定问之前能不能省事。

> `[crates/goose/src/permission/permission_inspector.rs:163-177]()` — 决策主干（完整函数见 144-270）
> <!-- snippet-base64:ICAgICAgICAgICAgICAgICAgICAgICAgLy8gMS4gQ2hlY2sgdXNlci1kZWZpbmVkIHBlcm1pc3Npb24gZmlyc3QKICAgICAgICAgICAgICAgICAgICAgICAgaWYgbGV0IFNvbWUobGV2ZWwpID0gcGVybWlzc2lvbl9tYW5hZ2VyLmdldF91c2VyX3Blcm1pc3Npb24odG9vbF9uYW1lKSB7CiAgICAgICAgICAgICAgICAgICAgICAgICAgICBtYXRjaCBsZXZlbCB7CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgUGVybWlzc2lvbkxldmVsOjpBbHdheXNBbGxvdyA9PiBJbnNwZWN0aW9uQWN0aW9uOjpBbGxvdywKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICBQZXJtaXNzaW9uTGV2ZWw6Ok5ldmVyQWxsb3cgPT4gSW5zcGVjdGlvbkFjdGlvbjo6RGVueSwKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICBQZXJtaXNzaW9uTGV2ZWw6OkFza0JlZm9yZSA9PiB7CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIEluc3BlY3Rpb25BY3Rpb246OlJlcXVpcmVBcHByb3ZhbChOb25lKQogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICAgICAgICAgLy8gMi4gQ2hlY2sgZm9yIGEgcmVhZC1vbmx5IGFubm90YXRpb24gaW4gU21hcnRBcHByb3ZlIG1vZGUKICAgICAgICAgICAgICAgICAgICAgICAgfSBlbHNlIGlmIGdvb3NlX21vZGUgPT0gR29vc2VNb2RlOjpTbWFydEFwcHJvdmUKICAgICAgICAgICAgICAgICAgICAgICAgICAgICYmIHNlbGYuaXNfcmVhZG9ubHlfYW5ub3RhdGVkX3Rvb2wodG9vbF9uYW1lKQogICAgICAgICAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgICAgICAgICBJbnNwZWN0aW9uQWN0aW9uOjpBbGxvdwogICAgICAgICAgICAgICAgICAgICAgICB9 -->
> -->

**Auto / Chat 直接短路**：`GooseMode::Auto` 无脑 Allow（`permission_inspector.rs:161`），`Chat` 直接 `continue` 跳过（不产出任何裁决，由主循环在 Chat 模式下强制跳过工具，`agents/agent.rs:2360-2376`）。Auto 模式是"放弃权限治理"，信任一切——这是用户显式选择自由度的代价。

### 决策二：SmartApprove 的"负向缓存"递归学习

smart_approve 要解决的根本矛盾：**Agent 的体验（少打扰）与安全（不放过）**。goose 用"只缓存负向结论"来调和。

`cache_non_readonly_decision`（`permission_inspector.rs:22-34`）：只有当一个读型工具被 LLM 判为**非只读**时，才把它登记为 `AskBefore`。也就是说：

- 工具是只读的 → **不缓存**（下次还让 LLM 判，因为"能不能用"取决于这次参数）
- 工具非只读 → **缓存 AskBefore**（下次直接问，不再浪费一次 LLM 调用）

> `[crates/goose/src/permission/permission_inspector.rs:27-33]()` — 仅缓存负向决策
> <!-- snippet-base64:ICAgICAgICBpZiBpc19yZWFkb25seSB7CiAgICAgICAgICAgIHJldHVybjsKICAgICAgICB9CiAgICAgICAgaWYgbGV0IE9rKHRvb2xfY2FsbCkgPSAmY2FuZGlkYXRlLnRvb2xfY2FsbCB7CiAgICAgICAgICAgIHBlcm1pc3Npb25fbWFuYWdlcgogICAgICAgICAgICAgICAgLnVwZGF0ZV9zbWFydF9hcHByb3ZlX3Blcm1pc3Npb24oJnRvb2xfY2FsbC5uYW1lLCBQZXJtaXNzaW9uTGV2ZWw6OkFza0JlZm9yZSk7CiAgICAgICAgfQ== -->

意图：只读工具的"可放行"是**有条件的**（取决于参数），不能永久记忆；但"非只读工具要问"这一结论是**无条件的**，缓存它每次能省一次 LLM 往返。测试 `smart_approve_only_caches_negative_name_wide_decisions`（`permission_inspector.rs:373-394`）精确锁定了这个行为：判只读后缓存为 `None`，判非只读后缓存为 `AskBefore`。

**权衡**：这本质是"用一次 LLM 调用换未来多次的省心"，缓存粒度是**工具名级别**（不是 工具名+参数哈希 级别）。代价是误判会被放大——若某多用途工具这次判非只读，之后同类调用就被永久要求审批，可能过度打扰。它比 permission_store 的"上下文哈希"粒度粗，换来的是实现简单、无需维护哈希。

### 决策三：让 LLM 判断只读 vs 硬编码——授权给推理

goose 没有为每个工具硬编码"是否只读"，而是把判断交给一个独立、专门的 LLM 调用（`permission_judge.rs`）。这是权限系统里最有争议也最有意思的设计。

**为什么不硬编码**：工具 schema 的 `read_only_hint` 只能表达"静态唯一答案"，但同一个工具（如 `multipurpose`）在不同参数下可能是读也可能是写。硬编码无法表达"看参数再定" 这种细粒度。所以 googe 让模型基于**工具名 + 参数**现场推理。

**怎么做**：构造一个专用只读判定工具 `platform__tool_by_tool_permission`（`permission_judge.rs:41-89`），把待判请求打包成 JSON，让被判模型的"分身"（同一个 provider，独立一次 `complete` 调用）返回哪些 request ID 是只读的（`permission_judge.rs:145-185`）。

**安全细节（注入防护内建）**：prompt 里明确把请求内容标为"UNTRUSTED TOOL REQUEST DATA (JSON)"，警告模型不要服从请求里的指令（`permission_judge.rs:112-115`），系统提示（`prompts/permission_judge.md`）不断重申"名字、参数都是不可信数据"，并要求"如果不确定，就不要判为只读"（保守默认）。

> `[crates/goose/src/permission/permission_judge.rs:112-114]()` — 请求数据标记为不可信
> <!-- snippet-base64:ICAgICAgICAgICAgdmVjIVtNZXNzYWdlQ29udGVudDo6dGV4dChmb3JtYXQhKAogICAgICAgICAgICAgICAgIlVOVFJVU1RFRCBUT09MIFJFUVVFU1QgREFUQSAoSlNPTik6XG57cmVxdWVzdHN9IgogICAgICAgICAgICApKV0s -->

**权衡**：把安全判断托付给 LLM，等于既相信 model 有判断力，又承认它可能被注入。goose 的防护是**双层**：
1. prompt 隔离 + 保守默认（不确定就当写）；
2. 即便判错了，也**只影响"是否要多问一次"**，不会造成越权——因为判成只读的结果只是放行"被认定为只读"的操作，而真正危险的操作由注解层 / 用户规则层已经把守。

**替代方案代价**：全用硬编码注解，简单确定性强，但无法覆盖参数维度，大量误问或漏放唯一答案工具；成本是每个工具要人工维护注解。LLM 判定省了维护，代价是每次 smart_approve 启用会多一次模型调用延迟。goose 的选用理由是：**把"成本"放在模型推理上（可扩展），把"确定性"放在规则上（可控），两者以 用户规则 > 注解 > LLM 的顺序显式叠加**。

### 决策四：两层记忆——permission.yaml vs tool_permissions.json

goose 有两类持久化权限，分工完全不同：

1. **permission.yaml（PermissionManager）**：按工具名的用户级规则，分 `always_allow / ask_before / never_allow` 三类（`config/permission.rs:25-30`）。它是**权威的用户意愿**，也是 smart_approve 的缓存场所（`user` 与 `smart_approve` 两个命名空间，`config/permission.rs:40-41`）。配置损坏就 panic 拒启（`config/permission.rs:49-59`）——宁可起不来也不带病运行。

2. **tool_permissions.json（ToolPermissionStore）**：按"工具名 + 参数哈希"的调用级记忆，带过期时间（`permission/permission_store.rs:11-20`）。记录形如 `context_hash`（对 arguments 做 blake3 哈希）、`allowed`、`expiry`。`check_permission` 用 `名称:context_hash` 做键，去找未过期的记录（`permission_store.rs:79-90`）。

关系：**PermissionManager 是"规则引擎 + 缓存"，ToolPermissionStore 是"参数敏感的调用历史"**。前者在每次决策主干里即时读取（`get_user_permission` / `get_smart_approve_permission`），后者更细粒度但当前主链路上主要由落盘与查询支撑（`record_permission` / `check_permission`）。

`tool_permissions.json` 的写入用"临时文件 + 原子 rename"来防写坏（`permission_store.rs:63-77`），加载时顺手清理过期条目（`permission_store.rs:57-58`）。到期机制设计上允许"按工具名判级别"这条主链不依赖参数哈希，保持 decision 的高吞吐。

### 决策五：审批握手——异步的"请求→回填"协调

审批是异步的：主循环无法也没必要阻塞等用户在同一个调用栈里点确认。goose 用**两个互补的握手原语**解决：

**ToolConfirmationRouter（当前主审批通道）**：一个 `HashMap<request_id, oneshot::Sender<PermissionConfirmation>>`。审批流（`handle_approval_tool_requests`，`agents/tool_execution.rs:112-204`）为每个待批请求 `register` 一个 oneshot receiver，向前端发出 `action_required` 消息后 `await`；前端用户点选后回调 `deliver()`（`agents/agent.rs:1608-1611`）把 `PermissionConfirmation` 塞回 oneshot 通道。

```rust
let confirmation_rx = self.tool_confirmation_router.register(request.id.clone()).await;
let confirmation = confirmation_rx.await ...; // 阻塞直到前端回传
```

`register` 会顺带清理已关闭的旧条目防泄漏（`tool_confirmation_router.rs:22-25`）；`deliver` 找不到等待者 / 接收端已 dropped 都安全打日志（`tool_confirmation_router.rs:27-45`）。这套 oneshot 模型让审批天然并发安全、无超时复杂度，且能精确按 request_id 路由。

**ActionRequiredManager（更通用的 elicitation 握手）**：一组带 session 隔离的 mpsc 流 + pending oneshot。`request_and_wait` 把一次"需要用户输入"的请求挂回一条 per-(session, tool_call) 的流（`action_required_manager.rs:66-121`），`claim_response` 校验 session 归属后把应答回填（`action_required_manager.rs:123-153`）。额外带**超时**（`timeout` 包裹，`action_required_manager.rs:163-184`），且支持用户选择的 Accept/Decline/Cancel 三态（`action_required_manager.rs:13-18`）。它被扩展管理流程用到（`agents/extension_manager.rs:1851-1859`）。

**Why 两个？** 一个是"确认工具执行"（AllowOnce/AlwaysAllow/AlwaysDeny 二值语义 + 权限记忆），一个是"索要任意输入"（Accept/Decline/Cancel 三态 + 带 schema）。审批确认对应 tool 的三档权限记忆，elicitation 对应自由输入协商，职责不同故各自独立实现。

**审批的权限记忆副作用**（`agents/tool_execution.rs:181-199`）：用户选 `AlwaysAllow` → 写 user 级 `AlwaysAllow`；选 `AlwaysDeny` → 写 `NeverAllow`。这形成用户每次点击都在训练权限表的学习闭环。**安全 finding 的决策会被记录**（`tool_execution.rs:149-163`）——即便最终放行，审计日志里也留下 BLOCK/ALLOW 标记。

> `[crates/goose/src/agents/tool_execution.rs:181-185]()` — 审批结果的权限学习副作用（完整分支见 165-199）
> <!-- snippet-base64:ICAgICAgICAgICAgICAgICAgICBpZiBjb25maXJtYXRpb24ucGVybWlzc2lvbiA9PSBQZXJtaXNzaW9uOjpBbHdheXNBbGxvdyB7CiAgICAgICAgICAgICAgICAgICAgICAgIHNlbGYudG9vbF9pbnNwZWN0aW9uX21hbmFnZXIKICAgICAgICAgICAgICAgICAgICAgICAgICAgIC51cGRhdGVfcGVybWlzc2lvbl9tYW5hZ2VyKCZ0b29sX2NhbGwubmFtZSwgUGVybWlzc2lvbkxldmVsOjpBbHdheXNBbGxvdykKICAgICAgICAgICAgICAgICAgICAgICAgICAgIC5hd2FpdDsKICAgICAgICAgICAgICAgICAgICB9 -->

### 决策六：PromptInjectionScanner——规则为主 + ML 可选增强

核心权衡：**规则查询零成本、确定、无上下文窗口担忧；ML 泛化强但贵、要外部服务。goose 默认规则，ML 可配置。**

扫描对象**只针对 shell 工具的 `command` 参数**（`scanner.rs:136-143`），并连同最近对话上下文一起看（`scan_conversation`，取最近 10 条用户消息，`scanner.rs:224-262`）——因为它怀疑的不只是命令本身危险，还可能是**对话被注入后诱导调用危险命令**。

**规则层（patterns.rs）**：静态正则集合，30+ 条覆盖 8 类威胁（`patterns.rs:48-308`）：文件系统破坏（`rm -rf /`，`patterns.rs:50-62`）、远程代码执行（`curl|bash`，`patterns.rs:79-84`）、数据外泄（SSH key、`/etc/shadow`）、权限提升（sudoers、SUID）、网络隧道 / 反弹 shell 等。复杂度上刻意做**反误报**工程：`rm_rf_home_or_root` 的正则精确到不会误伤 `rm -rf ./build`、`rm -rf ~/Documents` 等合法场景（`patterns.rs:406-410`, `patterns.rs:442-465`），`log_manipulation` 不误伤 `/dev/null`（`patterns.rs:496-516`）。风险等级映射置信度：Critical=0.95 / High=0.75 / Medium=0.60 / Low=0.45（`patterns.rs:37-44`）。

**ML 增强层（classification_client.rs）**：可选接一个 **HuggingFace Inference Text Classification 规范**的外部分类端点（`classification_client.rs:7-21`），返回多标签分数后做软归一化，取 top label 的 injection 分数（`classification_client.rs:141-220`）。模型可用 `SECURITY_ML_MODEL_MAPPING` 或直接端点配置（`scanner.rs:89-123`）。若 ML 初始化失败**回退纯规则**（`security/mod.rs:112-135`）——强健降级。

**两路置信度融合（combine_confidences，`scanner.rs:264-285`）**：
- 工具内容明显安全（<0.3）→ 直接用工具分（不看上下文，防噪声）；
- 工具安全但上下文可疑 → 打 0.9 折提醒（弱化）；
- 两者都高（>0.8）且都超 → `max*1.05` 封顶 1.0（强化互相印证）；
- 其余 → 加权 `工具×0.8 + 上下文×0.2`（工具为主信号）。

**判定阈值可配**（默认 0.8，`scanner.rs:125-129`）。命中阈值 = `is_malicious`，SecurityManager 再叠一层 `confidence > threshold` 才真正 **BLOCK**（`security/mod.rs:158-196`）；低于阈值只 **LOG** 不打扰。`should_ask_user=true` 时 SecurityInspector 转成 `RequireApproval`（带 🔒 告警，`security_inspector.rs:27-33`），即恶意工具**不直接拒绝、转交给用户裁决**（兼顾误报时的人工放行权）。

> `[crates/goose/src/security/scanner.rs:269-276]()` — 两路置信度融合规则（函数见 264-285）
> <!-- snippet-base64:ICAgICAgICAvLyBJZiB0b29sIGlzIHNhZmUsIGNvbnRleHQgaXMgbm90IHRha2VuIGludG8gYWNjb3VudAogICAgICAgIGlmIHRvb2xfY29uZmlkZW5jZSA8IDAuMyB7CiAgICAgICAgICAgIHJldHVybiB0b29sX2NvbmZpZGVuY2U7CiAgICAgICAgfQoKICAgICAgICBpZiBjb250ZXh0X2NvbmZpZGVuY2UgPCAwLjMgewogICAgICAgICAgICByZXR1cm4gdG9vbF9jb25maWRlbmNlICogMC45OwogICAgICAgIH0= -->

### 决策七：AdversaryInspector——规则驱动的最小 LLM 审查器

和 PromptInjectionScanner 的"正则查命令"不同，AdversaryInspector 是**用户写规则、LLM 执行审查**的黑盒守卫，专防"agent 本身被劫持成 rogue"（`adversary_inspector.rs:311-315`）。

启用方式特殊：往配置目录放 **`adversary.md`**，里面可选 frontmatter `tools:` 选被审工具 + 规则正文（`adversary_inspector.rs:58-71`）。文件不存在则整体禁用（`is_enabled` 返回 false，`adversary_inspector.rs:384-386`）。**失败开放（fail-open）**：LLM 调用出错也默认放行（`adversary_inspector.rs:470-488`）——因为这是"额外的一层保险"，宁可放行也不阻断正常开发。

默认规则给出一条核心哲学：**"Err on the side of ALLOW — only block truly dangerous things"**（`adversary_inspector.rs:45-46`）。它必须判断的例子里"install packages / run tests / use git / modify files 都算正常开发"，只有真正外泄 / 破坏 / 提权才 BLOCK。

审查基于：用户原始任务 + 最近 4 条用户消息 + 工具调用 + 规则（`adversary_inspector.rs:320-328`），要求 LLM 首行回 `ALLOW` 或 `BLOCK`（`adversary_inspector.rs:306-309`）。命中 BLOCK → Deny（配 🛡️ 前缀原因，`adversary_inspector.rs:461-468`）。

**权衡**：它和 PermissionInspector 的 LLM 判定不同——一个是"帮你省审批"（判只读），一个是"最后把关"（判危险）。前者默认放行追求体验，后者默认放行因为它是可选的额外保险。同样都靠同一 provider 的独立一次完整调用，代价是每次被审工具多一次模型往返 —— 所以**默认只在 shell 类工具上启用**（`DEFAULT_TOOLS`，`adversary_inspector.rs:14`）。

### 决策八：EgressInspector 与分类服务——日志先行，不阻断

EgressInspector **只检测、只记录、永远 Allow**（`egress_inspector.rs:317-387` 里 action 恒为 Allow），不做拦截。它把命令里的 URL、git remote、S3/GCS bucket、scp/ssh/docker registry 目标全挖出来（`egress_inspector.rs:49-187`），再判方向（outbound/inbound，`egress_inspector.rs:211-273`），最后作为审计事件落日志（`egress_inspector.rs:356-366`）。它是**可观测性层**而非安全门，所以在审批决策里不改变结论。

> `[crates/goose/src/security/egress_inspector.rs:369-371]()` — egress 只 Allow 不拦截（完整 InpectResult 见 369-383）
> <!-- snippet-base64:ICAgICAgICAgICAgcmVzdWx0cy5wdXNoKEluc3BlY3Rpb25SZXN1bHQgewogICAgICAgICAgICAgICAgdG9vbF9yZXF1ZXN0X2lkOiB0b29sX3JlcXVlc3QuaWQuY2xvbmUoKSwKICAgICAgICAgICAgICAgIGFjdGlvbjogSW5zcGVjdGlvbkFjdGlvbjo6QWxsb3cs -->

它依赖的 ClassificationClient 则是**独立的外部分类服务**，被 scanner 用于 ML 层（见决策六），一套客户端两处复用（命令分类 + prompt 分类，`scanner.rs:64-83`）。

---

## 全局关联：四类决策如何汇成一个策略

把以上决策放回"goose 如何决定一次工具调用是否被允许"的总问题，可以抽象成**责任的同心分层**：

| 层 | 组件 | 回答的问题 | 失败代价 |
|----|------|-----------|---------|
| 硬规则 | PermissionManager user 配置 + 扩展管理特判 + 只读注解 | "用户/作者明说要不要" | panic 拒启 / 强审批 |
| 代价优化 | LLM 只读判定 + smart_approve 负向缓存 | "这次调用能省一次打扰吗" | 多一两次误问 |
| 输入防护 | PromptInjectionScanner（正则+ML） | "命令/上下文是不是被注入" | 误报被用户人工放行 |
| 兜底卫士 | AdversaryInspector | "agent 是不是已经 rogue" | 多一次模型延迟 |
| 审计 | EgressInspector | "数据往哪走" | 无（纯记录） |
| 人肉裁决 | ToolConfirmationRouter + ActionRequiredManager | "人类最终怎么定" | 需要用户在场 |

这套分层体现出 goose 的安全哲学是**"纵深防御 + 体验优化 + 人类最终决策权"三者的折中**：

- **纵深**：同一工具可能同时过正则扫描、LLM 只读判定、Adversary LLM 审查三层，任何一层拒绝都算拒绝。
- **体验**：Auto 全放 / SmartApprove 靠注解与 LLM 帮用户省点击，负向缓存保证"省"有记忆。
- **人类决定权**：即便 [SecurityInspector 判定恶意，也只转成 `RequireApproval`（带告警）而非硬拒（`security_inspector.rs:27-33`）；Adversary 的 LLM BLOCK 才硬拒。**"机器告警、人来做主"** 贯穿始终——除了用户显式 NeverAllow 和配置损坏这类硬死线，其余都是"问一下人"。

**本机 agent 的信任边界**：对类似 goose 这样的本地 Agent，安全模型假定**代理进程可信、模型输出不可信、工具入参不可信**。权限模块保障"代理只能访问用户授权的工具"，安全模块保障"不可信输入不能指挥代理做危险动作"。因为没有多租户隔离诉求（单用户本机），所以采用"问人"而非彻底的权限分离，这是与 server 型 agent 本质不同的安全设计取向。

---

## 权衡与替代方案的代价总结

| 决策点 | goose 选择 | 为什么 | 放弃的替代方案及其代价 |
|--------|-----------|--------|------------------------|
| 只读判定 | LLM 每次按参数推理 | 工具只读性依赖参数，硬编码表达不了 | 全硬编码注解：确定但无法表达参数维度，需人工维护，且误问率高 |
| smart_approve 记忆 | 只缓存"非只读"（工具名级） | 只读结论有条件不可永久记忆；负向结论无条件可缓存省调用 | 缓存"只读结论"：省更多但会把条件性放行永久化，参数变了也放行，风险高 |
| 恶意判定 | 正则为主 + 可选 ML | 正则零成本确定；ML 泛化强但要外部服务 | 全 ML：成本高、依赖服务可用性；goose 用 fail-open 回退保住线 |
| 恶意工具处置 | 转 RequireApproval 而非硬拒 | 保留误报时用户人工放行权 | 直接 Deny：更严但误报会卡死正常开发，与"err on the side of ALLOW"哲学冲突 |
| Adversary 审查 | 失败开放 + 默认只查 shell 类 | 它是可选保险，不能阻断正常开发；只查高危面控成本 | 失败关闭 + 查所有：更安全但开发体验受损，且 LLM 审查稳定性不可靠时会频繁中断 |
| 配置损坏 | panic 拒启（yaml）/ 原子写（json） | 权限是安全边界，带病运行比拒绝启动更危险 | 静默回退默认：可能让危险工具被误放行 |
| 审批握手 | oneshot+可超时 + action_required 流 | 异步不阻塞主循环，天然并发安全 | 同步阻塞式审批：实现简单但主循环卡死，无法多轮流式交互 |

---
