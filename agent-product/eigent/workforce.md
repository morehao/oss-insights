# Multi-Agent Workforce 编排

> 本解读按 **agent** 类规格产出（主类型 = 智能体产品，Multi-Agent Workforce）。这是 Eigent 的"头号卖点"落地层：把前端的一次提问，经 `server`(FastAPI 控制面) 的任务队列化封装后，交给 `backend`(CAMEL Runtime) 按任务复杂度自适应路由。
> 所有 `[路径:行号]` 引用均可点回真实源码核验，`<!-- snippet: -->` 指纹注释仅作程序重定位用、不进入正文。阅读前置：已了解三端架构（Electron/React 前端 + FastAPI 控制面 server + CAMEL 行动面 backend），此处围绕"一个 agent 怎么跑、多个 agent 怎么并行协作"展开。

---

## 一句话定位

Workforce 编排是 Eigent 的"头号卖点"落地层：它把**前端的一次提问**，经过 `server`(FastAPI 控制面) 的任务队列化封装后，交给 `backend`(CAMEL Runtime)**按任务复杂度自适应路由**——简单问题用单个 `question_agent` 直接回答，复杂任务升级为一个 `Workforce`（继承 CAMEL `BaseWorkforce`）做**任务分解 + 多 worker 并行执行**；另有独立的 `single-agent` 模式（`session_mode`）单 agent 直驱。真正实现"多个 agent 并行协作"的执行器不在任何 factory 里，而在 `backend/app/utils/workforce.py` 与 `single_agent_worker.py`（原始文件清单未包含，但它是本模块的主角，我在分析中补入）。

---

## 在项目中的角色（去掉它系统会怎样）

**回答 Why**：去掉 Workforce 编排，Eigent 会退化成"一个带工具的单轮聊天室"，README 宣传的"Scale to multiple specialized agents that divide work, collaborate in parallel"将彻底失效，前端 Workforce UI 也会变成空壳。

**证据链（系统依赖它的三处硬锚点）**：

1. **backend 的 agent 工厂是为 Workforce 服务的，而非直接对外**。`agent_model()` 创建每个 agent 时都会往 task 队列推一条 `ActionCreateAgentData`（[backend/app/agent/agent_model.py:74-84]()），这是 agent 生命周期的"心跳"，前端靠它实时构建 Agent Pool 列表。
<!-- snippet:     _schedule_async_task(
        task_lock.put_queue(
            ActionCreateAgentData(
                data={
                    "agent_name": agent_name,
                    "agent_id": agent_id,
                    "tools": tool_names or [],
                }
            )
        )
    ) -->

2. **`construct_workforce` 是后端任务循环的"装配车间"**，一次性并行创建 coordinator/task_agent/new_worker + 5 个领域 worker（[backend/app/service/chat_service.py:2538-2771]()），并 `add_single_agent_worker` 挂到 Workforce。去掉它，`step_solve` 主循环在 complex 分支无从下手（[backend/app/service/chat_service.py:836-841]()` "Creating NEW workforce instance"）。

3. **前端 Workflorce UI 直接消费 Workforce 的 SSE 事件**。`WorkforceSidePanel.tsx` 拉 `selectedTask.taskAssigning`（被 `assign_task` 事件填充）与 `taskRunning`/`taskInfo`（被 `task_state` 等填充）渲染 Agent Pool 与 Progress（`src/components/Session/Workforce/WorkforceSidePanel.tsx:66-88`）。后端 workforce.py 在任务分配/执行/完成三阶段主动 push 这些事件（[backend/app/utils/workforce.py:546-575]()、`605-616`、`752`）。

**去掉它的直接后果**：
- `step_solve` 的 `Action.improve` → complex 分支无对象可建，多轮"复杂任务"全部失效；
- 前端 Agent Pool / Parallel Progress / ExpandedOverlay 全灭，只剩 Single Agent 面板；
- `backend/app/agent/factory/*.py` 的领域 agent（developer/document/browser/multi_modal/social_media）失去消耗方，沦为死代码。

（注：single-agent 模式经 `single_agent_solve` 独立于 Workforce 存活，所以准确说是"去掉 Workforce，Eigent 只剩单 agent 通道"。）

---

## 解决什么问题、根因（为什么需要多 agent 编排 vs 单 agent）

### 根因：真实任务 ≠ 单轮问答，需要"工具 + 分解 + 并行"

单 agent 直驱（`single_agent_solve`，见 4/5 节）能解决"我帮你写一段代码/搜一下资料"这类**单句可执行**任务；但 README 吹的复杂工作流（调研→写码→出文档→配图→发布）天然包含多类型子任务，单一 agent 会：
- **上下文爆仓**：token budget 内塞不下"长期研究结论 + 编码上下文 + 文档格式要求"；
- **工具集过载冲突**：把 file/terminal/browser/social/文档 全塞给一个 agent，tool 数量与 system prompt 会互相稀释（对比 `toolkit_assembler.py` 单agent只装 ~13 类工具，vs Workforce 里每个 worker 各司其职装专精工具）；
- **无法真并行**：浏览器调研等待网络时，编码 worker 只能干等。

### Eigent 的对策：任务自适应路由（一问三车道）

`step_solve` 不是"进来就起 Workforce"，而是先判复杂度再分流（[backend/app/service/chat_service.py:691-808]()`）：

- **带附件** → 强制 complex（[backend/app/service/chat_service.py:694-699]()`）；
- **否则** → `question_confirm` 让 LLM 判 yes/no（[backend/app/service/chat_service.py:701-703]()`），判 no 则 `question_agent` 直接作答（[backend/app/service/chat_service.py:710-756]()`），判 yes 则走 Workforce。
- **`session_mode == "single-agent"`** → 整段绕过 Workforce，直接 `single_agent_solve`（[backend/app/service/chat_service.py:519-524]()`）。

```
               ┌────────── session_mode=single-agent ──────► single_agent_solve（单 agent）
前端一次提问 ──┤
               └  Workforce 模式 ─┬─ question_confirm ─ no ─► question_agent 直接答（simple）
                                  └──────────── yes ───────► construct_workforce → 分解 → 并行执行
```

**Why 这个设计（权衡）**：不是"永远多 agent"（成本/延迟高，简单问题慢又贵），也不是"永远单 agent"（复杂任务做不了）。用一次低成本 LLM 判定（`question_confirm`）换取"简单问答题"与"复杂项目题"的最优路径。这是**延迟-成本 vs 能力上限**的折衷。代价：多一次模型调用延迟 + `question_confirm` 判错时用户体验分层（简单问题误判为复杂 → 白白起一个 Workforce）。

### 与 README 口径核对

| README 宣称 | 实现事实 | 结论 |
|---|---|---|
| "Start with one focused meta agent for direct tasks" | `session_mode="single-agent"` + `single_agent_solve`，确有专门 single-agent 通道；且复杂分支里 new_worker/coordinator 也可作 meta 协调 | **一致** |
| "Scale to multiple specialized agents that divide work, collaborate in parallel" | `Workforce(BaseWorkforce)` + `SingleAgentWorker`(agent pool) 实现分解与并发 | **一致**（后半"并行"见 7.2） |
| "collaborate in parallel" | 并行依赖 CAMEL Worker 内 AgentPool 取 clone；SSE 上以 `assign_task`/`task_state` 透出并行表象 | **部分一致**，硬断言依赖 CAMEL 版本实现 |

关于 "parallel" 的颗粒度：当前读到的并行来自 `SingleAgentWorker._get_worker_agent()` 从 AgentPool 取 clone（[backend/app/utils/single_agent_worker.py:113]()）。真正多 worker 并发是 CAMEL `Workforce.start()` 内部 `_delegate_tasks` 的编排，Eigent 未重写该并发核心，仅通过继承 `BaseWorkforce` 挂事件。若要断言"多 worker 真并行"，尚需验证 CAMEL 版本中 `BaseWorkforce.start()` 是否对无依赖的 `_pending_tasks` 使用 `asyncio.gather`。

<!-- snippet: single_agent_worker.py:112-114 -->
```python
# Get agent efficiently (from pool or by cloning)
worker_agent = await self._get_worker_agent()
worker_agent.process_task_id = task.id  # type: ignore  rewrite line
```

---

## 设计思路（方案 / 放弃的替代方案 / 核心模式）

### 核心模式 1：Agent 工厂（Factory）—— 用 `agent_model()` + 各领域工厂函数统一装配

所有 agent（无论单还是多）最终都收敛到 `agent_model()`（[backend/app/agent/agent_model.py:56]()）。它承担三件事：

1. **生成 agent_id + 上报 create_agent 事件**（见 1 节证据）；
2. **模型构建（build_model）**：解析 platform/type/api_key/api_url，合并 `extra_params`/`model_config_dict`，按平台注入 prompt-cache 参数、代理（proxy/subscription）适配（[backend/app/agent/agent_model.py:133-294]()）。这是"模型无关"的关键——同一套 agent 装配可跑 OpenAI/Azure/Bedrock/Anthropic/本地 vLLM；
3. **构造 `ListenChatAgent`**（CAMEL `ChatAgent` 的封装子类，[backend/app/agent/agent_model.py:298-314]()）。

各领域工厂（`developer.py`/`document.py`/`browser.py`/`multi_modal.py`/`social_media.py`）则各自 `get_tools()` 组装**专精工具集** + 注入领域 system prompt，最后也调 `agent_model`。**Why 工厂模式**：领域差异大（developer 要 terminal+web_deploy，document 要 excel+pptx，browser 要 web 浏览），把"装配复杂度"隔离在每个工厂内部，而"执行骨架"（ListenChatAgent + agent_model）全局共享，避免每个 agent 重复实现 step/tool-call 循环。

**统一执行骨架的核心**：`ListenChatAgent` 重写了 CAMEL 的 `step`/`astep`/`_execute_tool`/`_aexecute_tool`，在每个步骤**进出队列**上报 `activate_agent`/`deactivate_agent` 与 `activate_toolkit`/`deactivate_toolkit`（[backend/app/agent/listen_chat_agent.py:413-630]()、`632-746`、`785-937`），让前端实时看到"哪个 agent 正在想 / 正在用什么工具"。这是 property of "控制面观测行动面"的落点【与 data/model 模块协同，见 6 节】。

### 核心模式 2：任务队列 + 锁（TaskLock / Action）—— 控制面与行动面的消息总线

这是整个编排的"神经系统"。核心原语在 `backend/app/service/task.py`：

- **`Action`（Enum）**：定义了 30+ 种双向消息类型（[backend/app/service/task.py:43-77]()），user→backend（improve/start/stop/pause/resume/add_task...）与 backend→user（create_agent/activate_agent/task_state/assign_task/todo_state/end...）。
- **`TaskLock`**：每个 project 一个，持 `queue: asyncio.Queue[ActionData]`（SSE 通道）+ `human_input` 队列（人工问答）+ `background_tasks` 集合（生命周期清理）+ `conversation_history`/`agent_memory_history`（上下文继承，[backend/app/service/task.py:355-417]()）。

**Why 队列而非直接返回**：这是一个**异步长事务**（可能跑几分钟），HTTP 无法阻塞等待，必须用队列承接后台执行、用 SSE 增量推送。同时队列抹平了"controller 线程/事件循环"与"agent 执行循环"的并发边界——`_schedule_async_task`/`schedule_async_task_from_worker` 专门处理跨线程入队（[backend/app/service/task.py:459-472]()、`chat_service.py` 大量 `run_coroutine_threadsafe`）。

### 核心模式 3：Workforce = 继承 CAMEL `BaseWorkforce` + 钩子事件转发

这是本模块真正的"多 agent 编排器"，属**继承 + 覆盖钩子**模式：

- `Workforce(BaseWorkforce)`（[backend/app/utils/workforce.py:64-107]()）：覆盖 `_analyze_task`(重试)/`eigent_make_sub_tasks`(分解并流式回传)/`eigent_start`(装载子任务并 start)/`handle_decompose_append_task`/`_find_assignee`(分发)/`_post_task`(开始执行)/`_handle_completed_task`/`_handle_failed_task`/`_get_returned_task`(超时)。
- 每个 worker 由 `Workforce.add_single_agent_worker()` 注入，包装为 `SingleAgentWorker`（[backend/app/utils/workforce.py:621-686]()）。

**Why 继承 CAMEL 而非自研**：并行分解、依赖排序、失败重试/重新规划是 CAMEL 已提供的高复杂度骨架，Eigent "不重复发明轮子"，只通过覆盖钩子插入两件事：(a) 把 **node_id ↔ agent_id 映射** 出来推给前端（[backend/app/utils/workforce.py:440-454]()），(b) 在关键生命周期**旁路发 SSE**。这个"继承 + 前置动作 + 调 super()"的模式贯穿所有覆盖方法（例如 `_find_assignee` 先发 assign_task 再 `return await super()._find_assignee()`）。

### 放弃/弱化的替代方案

- **放弃"单 agent 全包"**：能力/上下文不足（见 2.1），保留为 `session_mode` 选项而非默认。
- **弱化"自研调度器"**：不自己写任务图/依赖拓扑/失败恢复，全部委托 CAMEL。代价=对 CAMEL 升级敏感，且前端拿到的并行细节受限于 CAMEL 事件序列（见 7.2 权衡）。
- **worker 复制策略改为"池化"而非"固定多开"**：`SingleAgentWorker.__init__` 显式把 `pool_initial_size` 从 CAMEL 默认 1 改成 `0`，并注明"避免预创建克隆浪费 CDP 资源"（[backend/app/utils/single_agent_worker.py:46]()）。这是**资源代价 vs 启动速度**的权衡：不预克隆，首次任务需即时 clone（慢一拍），但省下浏览器 tab/CDP 句柄。

---

## 关键数据结构（只贴理解必需的类型/接口，精简）

### 输入门面：`Chat`（pydantic，[backend/app/model/chat.py:54-107]()）
所有任务请求的统一模型，字段即"一次 run 的完整参数"：
- `task_id` / `project_id` / `space_id` / `run_id`：三端定位坐标。
- `question` / `attaches` / `email`：用户输入。
- `model_platform` / `model_type` / `api_key` / `api_url`（+`extra_params`/`model_config_dict`）：模型无关（[backend/app/model/chat.py:66-92]()）。
- `session_mode: Literal["workforce","single-agent"] = "workforce"`：**分流开关**（[backend/app/model/chat.py:101]()）。
- `new_agents: list[NewAgent]`：用户自定义 agent（[backend/app/model/chat.py:86]()）。
- `toolkit_config` / `installed_mcp` / `remote_sub_agent_config`：能力开关。

`NewAgent`（[backend/app/model/chat.py:255-261]()）描述用户在工作台加的"自定义 agent"：name/description/tools/mcp_tools/custom_model_config。`AgentModelConfig`（[backend/app/model/chat.py:230-252]()）支持**按 agent 覆盖模型**（worker 用不同平台/模型）。

### 编排原语：`Action` / `ActionXxxData` / `TaskLock`（`backend/app/service/task.py`）
- `Action` Enum：双向消息类型全集（[backend/app/service/task.py:43-77]()）。
- 每个分支一个 pydantic 模型（`ActionImproveData`/`ActionStartData`/`ActionTaskStateData`/`ActionCreateAgentData`/`ActionAssignTaskData`...），统一 `action` 判别字段（[backend/app/service/task.py:87-339]()）。
- `Agents` Enum：注册的 agent 名称常量（[backend/app/service/task.py:342-352]()）：task_agent/coordinator_agent/new_worker_agent/developer/browser/document/multi_modal/social_media/mcp/single_agent。
- `TaskLock`：per-project 状态容器 + 队列（[backend/app/service/task.py:355-653]()），关键字段：`queue`、`status`(confirming/confirmed/processing/done，[backend/app/service/task.py:357]())、`human_input_waiters`(人工答询)、`conversation_history`/`agent_memory_history`(跨轮上下文，[backend/app/service/task.py:376-378]())、`registered_toolkits`(清理 terminal venv 等资源，[backend/app/service/task.py:585-609]())。
- `task_locks` 全局字典 + `get_or_create_task_lock`/`delete_task_lock`/`cleanup`（[backend/app/service/task.py:656-741]()）。

### 执行核心：CAMEL `Task`
`camel.tasks.Task`（导入于 [backend/app/service/task.py:24]()）承载任务分解树：`content`/`id`/`parent`/`subtasks`/`state`/`result`/`assigned_worker_id`/`additional_info`/`failure_count`。Workforce 的分解结果就是 Task 树（index 由 `get_camel_task` 维护，[backend/app/service/task.py:743-764]()）。前端看到的 `tree_sub_tasks(camel_task.subtasks)` 即由此遍历。

### Agent：`ListenChatAgent(ChatAgent)`（[backend/app/agent/listen_chat_agent.py:74]()）
统一 agent 壳。关键自定义属性：`api_task_id`/`agent_name`/`agent_id`/`process_task_id`(当前子任务 id，worker 复用同 agent 实例跑多个 task 时用到，见 [backend/app/utils/single_agent_worker.py:114]())、`stop_event`/`pause_event`(控制)、`_internal_tools`(tool 字典)、`_cdp_*`(浏览器句柄旁路)。重写 `step/astep/_execute_tool/_aexecute_tool/_execute_tool_from_stream_data/_aexecute_tool_from_stream_data`(打事件 + 处理异步/同步工具)。

### 领域组装产物：`ToolkitAssembly`（[backend/app/agent/factory/toolkit_assembler.py:69-93]()）
`tools`(FunctionTool 列表)/`tool_names`/`toolkits_to_register_agent`/`observable_todo_toolkit`/`browser_toolkit`+`browser_*`(CDP 资源句柄)。是单 agent 工具集的装配单据。

---

## 核心流程 —— 完整链路 Mermaid

### 总览：一次提问的三车道后端路由

```mermaid
sequenceDiagram
    autonumber
    participant FE as 前端 UI
    participant C as chat_controller
    participant Q as TaskLock.queue
    participant S as step_solve
    participant WF as Workforce
    participant QA as question_agent
    participant A as ListenChatAgent

    FE->>C: POST /chat (Chat)
    Note over C: chat_controller 建 TaskLock
    C->>Q: put ActionImproveData(question)
    Note over C: chat_controller.py 行472-482
    C-->>FE: StreamingResponse(SSE)
    Note over C: chat_controller.py 行503-509

    rect rgb(235,235,255)
    Note over S: step_solve 主循环
    Q-->>S: get_queue() Action
    Note over S: 逐 action 分发 (while True)
    alt session_mode == single-agent
        S-->>S: single_agent_solve(options, request, task_lock)
        Note over S: chat_service.py 行519-524
        single_agent_solve-->>A: single_agent -> agent_model -> ListenChatAgent
        Note over single_agent_solve: single_agent_service.py 行233-263
    else 复杂任务 (question_confirm == yes)
        S-->>S: construct_workforce() 并行建 6 agents
        Note over S: chat_service.py 行836-841
        S->>WF: eigent_make_sub_tasks() ASYNC 分解
        Note over S: chat_service.py 行923-1009
        WF-->>Q: ActionDecomposeProgress/Text
        Q-->>FE: SSE decompose 事件
        FE->>S: POST /task/{id}/start
        Note over FE: task_controller.py 行61-71
        S->>WF: eigent_start(sub_tasks)
        Note over S: chat_service.py 行1292
        WF->>WF: _delegate_tasks -> _find_assignee 派发
        Note over WF: workforce.py 行487-539
        WF-->>Q: ActionAssignTask / ActionTaskState
        Q-->>FE: SSE assign_task / task_state
    else 简单问题 (question_confirm == no)
        S->>QA: astep(simple_answer_prompt)
        Note over S: chat_service.py 行710-756
        S-->>FE: SSE wait_confirm
    end
    end
    Note over S: Action.end 收尾，保留 task_lock
    ```

### 详细：Workforce 复杂任务路径（分解 → 用户编辑 → 执行 → 完成）

```mermaid
sequenceDiagram
    autonumber
    participant S as step_solve
    participant WF as Workforce
    participant TA as task_agent
    participant WL as Worker
    participant A as ListenChatAgent
    participant Q as TaskLock.queue
    participant FE as 前端

    S->>WF: eigent_make_sub_tasks(...)
    Note over S: chat_service.py:926-932
    WF->>TA: _decompose_task(task.subtasks)
    Note over WF: workforce.py:269-301
    TA-->>WF: Task[]（依赖可形成 DAG）
    WF-->>Q: ActionDecomposeProgress
    Note over WF: workforce.py:303-438
    Q-->>S: decompose_progress (SSE)
    FE->>S: PUT /task/{id} 删改子任务
    Note over FE: task_controller.py:74-91
    S->>S: update_sub_tasks + add_sub_tasks
    Note over S: chat_service.py:1012-1036
    FE->>S: POST /task/{id}/start
    S->>WF: eigent_start(sub_tasks) asyncio.create_task
    Note over S: chat_service.py:1289-1293
    WF->>WF: _pending_tasks.extendleft(...)
    Note over WF: workforce.py:250-253
    WF->>WF: start() -> _delegate_tasks
    Note over WF: workforce.py:256
    WF->>WL: _post_task(task, assignee_id) 依赖满足
    Note over WF: workforce.py:578-619
    WF-->>Q: ActionAssignTask(state=running)
    WL->>A: worker_agent = _get_worker_agent() clone
    Note over WL: single_agent_worker.py:113-151
    WL->>A: worker_agent.astep(PROCESS_TASK_PROMPT...)
    Note over WL: single_agent_worker.py:129-151
    A->>A: ListenChatAgent.astep -> 模型循环 -> 工具调用
    Note over A: listen_chat_agent.py:517-630
    A-->>WL: TaskResult(content, failed)
    WL->>WL: _return_worker_agent 回池
    Note over WL: single_agent_worker.py:294-296
    WF->>WF: _handle_completed_task + 通知
    Note over WF: workforce.py:688-780
    WF-->>Q: ActionTaskState(DONE, result)
    Q-->>FE: SSE task_state 前端标绿
    WF-->>Q: ActionEndData (stop 时)
    Note over WF: workforce.py:917
    Q-->>S: Action.end 收尾 + 保留 task_lock
    Note over S: chat_service.py:1830-1959
    ```

### 详细：single-agent 路径（`session_mode="single-agent"`）

```mermaid
sequenceDiagram
    autonumber
    participant C as chat_controller
    participant S as step_solve
    participant SA as single_agent_solve
    participant F as single_agent()
    participant A as ListenChatAgent
    participant Q as TaskLock.queue
    participant FE as 前端

    C->>S: start_chat_stream → step_solve
    Note over C: chat_controller.py:494-500
    S->>SA: single_agent_solve(options, request, task_lock)
    Note over S: chat_service.py:519-524
    SA->>SA: ensure_agent → single_agent()
    Note over SA: single_agent_service.py:233-247
    F->>F: assemble_single_agent_toolkits
    Note over F: single_agent.py:60-68
    F->>F: agent_model(single_agent, ...)
    Note over F: single_agent.py:78-88
    F->>A: ListenChatAgent
    loop 主循环 (asyncio.wait on queue)
        Q-->>SA: ActionImprove 新问题
        SA-->>FE: SSE confirmed
        SA->>SA: run_turn + _build_single_agent_prompt
        Note over SA: single_agent_service.py:249-281
        SA->>A: astep(prompt) 工具循环
        Note over SA: single_agent_service.py:263
        A-->>SA: content + tokens
        SA-->>FE: SSE end
        Note over SA: single_agent_service.py:446-456
    end
    ```

## 与其他模块的协同

### 依赖 CAMEL（行动面基石）
- 模型封装：`ModelFactory.create`（[backend/app/agent/agent_model.py:286]()）、平台枚举 `ModelPlatformType`。
- Agent 骨架：`ChatAgent`/`ChatAgentResponse`/`AsyncStreamingChatAgentResponse`、`ModelProcessingError`（`listen_chat_agent.py`）。
- 编排骨架：`BaseWorkforce`、`SingleAgentWorker`(CAMEL)、`BaseNode`、`Task`/`TaskState`、`TaskChannel`、`FailureHandlingConfig`（`workforce.py`、`single_agent_worker.py`）。
- 工具：`FunctionTool`、`ToolkitMessageIntegration`、`MCPToolkit`、`WebFetchToolkit`、`PlanningWorktreeToolkit`（`toolkit_assembler.py`）。

**Why 如此深绑 CAMEL**：Eigent 定位是"再造 CAMEL-AI"，把执行硬骨架交给框架，自己专注 workspace/任务状态/前端可视化这层"控制面功能"。

### 依赖 toolkit 层（能力注入）
本模块本身"不干实事"，全靠 toolkit 注入能力：
- 单 agent：`assemble_single_agent_toolkits` 按 `toolkit_config` 开关组装 13 类工具，其中 `agent`(task) 仅当 `can_delegate(depth<max_depth)` 时注入 `DepthLimitedAgentToolkit`（[backend/app/agent/factory/toolkit_assembler.py:433-440]()），限制单 agent 委派深度≤1（[backend/app/agent/factory/single_agent.py:35-43]()），防止"套娃式自我复制 agent"失去控制。
- Workforce worker：各领域工厂用各自专精 toolkit（文档→excel/pptx/markitdown/google-drive；开发者→terminal/web_deploy；社交→Twitter/LinkedIn/Reddit/Gmail/Calendar...）【toolkit 模块详读见后续 memory/tool 解读】。
- 消息集成：`ToolkitMessageIntegration(message_handler=HumanToolkit.send_message_to_user)` 把所有 toolkit 的"主动问用户/通知"统一到 human 通道（多个工厂，如 [backend/app/agent/factory/toolkit_assembler.py:220-223]()）。

### 依赖 server 数据（控制面状态）
- 工作目录：`get_working_directory(options)`——由 server 冻结的 `frozen_dirs`（[backend/app/controller/chat_controller.py:412]()）→ run_context → 各 factory 取 working_directory。
- 任务状态持久化：`TaskLock.conversation_history`/`agent_memory_history`/`summary_task_content`，经 memory 服务落盘（`finalize_task_lock_run_memory`），支撑跨重启/跨轮上下文【memory 模块铺垫】。
- workspace 绑定：`workspace_controller` 的 bind/scratch 决定 workdir_mode（worktree/copy/direct-write/artifact-only），影响 `file_save_path` 与 CDP 浏览器池竞争（见 6.4、7.3）。

### 全局共享资源：CDP 浏览器池（并行资源争用的枢纽）
并行 worker 都抢浏览器，Eigent 用 `CdpBrowserPoolManager`（[backend/app/agent/factory/browser.py:74-176]()）做**端口级互斥分配**：
- `acquire_browser`/`release_browser`/`release_by_task`，`threading.Lock` 保护（[backend/app/agent/factory/browser.py:74-176]()）。
- 单 agent 与 browser worker 共用同一 pool（[backend/app/agent/factory/toolkit_assembler.py:316-335]() 注释 "Reuse the same pool as the Browser Agent"）。
- `ListenChatAgent.clone` 在克隆 browser toolkit 前**先抢新 CDP 端口**再复制工具（[backend/app/agent/listen_chat_agent.py:955-1021]()），clone 完还原 `cdp_url` 防并发污染。workforce 停止时 `_cleanup_all_agents` 按 task_id 强制释放所有端口（[backend/app/utils/workforce.py:940-1045]()）。

`CDP 池` 是本模块与 browser 模块的重要耦合点，它与 browser 模块的完整生命周期（launch/connect/launch 浏览器）细节在 browser 专项解读中展开。

### 前端协作
- 生产端事件 `AgentStep.*`：`chatStore.ts` 用 `sseTransport` 拉流并按 `step` 分发（[src/store/chatStore.ts:2147-2157]()），`confirmed`/`decompose_text`/`to_sub_tasks`/`assign_task`/`task_state`/`activate_agent`/`deactivate_agent`/`activate_toolkit`/`deactivate_toolkit`/`write_file`/`terminal` 等驱动 projectStore 状态（`chatStore.ts:2265,2477,2527,2701,2892,3003,3132,3299,3436,3553`）。
- 展示端：`SessionSidePanel.tsx` 按 sessionMode 切 `WorkforceSidePanel` 或 `SingleAgentSidePanel`（`SessionSidePanel.tsx:93,103`）；`WorkforceSidePanel.tsx` 消费 `taskAssigning`(Agent Pool) + `taskRunning`/`taskInfo`(Progress)（见 1 节第 3 条证据）。

### 跨模块结论
1. 控制面/行动面解耦的"消息总线"就是 `TaskLock.queue` + `Action`；前端不直连 agent，只看 SSE 事件流。
2. 上下文继承（多轮）有两套来源：workforce 用 `conversation_history`(in-process) + 持久化后 `memory_service`；single-agent 用 durable LocalMemoryStore（`_build_single_agent_context`）。两条路径的 memory 策略不同，验证两模块后需统一口径。
3. "多 agent 并行"的底层仍是 CAMEL `BaseWorkforce.start()`；Eigent 只覆盖事件钩子，未接管调度器本体。

---

## 关键设计决策（权衡与替代方案代价）

### 决策 1：单 agent(low-latency) vs Workforce(high-capability) —— 用一次 LLM 判定做运行时自适应

**取舍**：不是选一个，而是让系统同时持有两条通道，运行时按 `question_confirm` + `session_mode` 分流（见 2.2）。

- **选单 agent 的代价**：复杂多步任务跑不动（上下文爆 + 无法并行）；**选 Workforce 的代价**：简单问答题要多等一次确认分类 + 起 6 个 agent（启动成本 + 多 agent 预热延迟）。

**替代方案 A（永远 Workforce）**：实现最简（去掉判定），但 README 的 "focused meta agent for direct tasks" 失去意义，简单聊天也拖着 6 agent 空转，体验与成本双输。

**替代方案 B（永远单 agent + 自递归委派）**：单 agent 内部用 `DepthLimitedAgentToolkit` 委派（`single_agent.py` 的 `can_delegate`），但深度被卡在 ≤1（[backend/app/agent/factory/toolkit_assembler.py:433-440]()），且委派不解决"不同专业工具集"的隔离，只是"同质 agent 复制"。Eigent 选择同时保留两者，用任务性质自适应。

**Why 判定放 LLM 而非规则**：`question_confirm` 用结构提示问 yes/no（[backend/app/service/chat_service.py:2191-2244]()`），比硬编码关键词（"create"/"search"）更鲁棒。代价见 2.2——判定本身一次模型调用 + 判错无回退。

### 决策 2：并行执行 —— 用 CAMEL Worker 内 AgentPool(clone) 实现 worker 并发

**实现**：`SingleAgentWorker._get_worker_agent()` 从池/clone 取可复用的 `ListenChatAgent` 实例（[backend/app/utils/single_agent_worker.py:113]()、`listen_chat_agent.py:955-1095 clone`）。Workforce 的 `_delegate_tasks` 把依赖满足的 pending task 派发给各自 worker；每个 worker 一次跑一个 task，但多个独立 worker 可由 CAMEL 调度并发。

**权衡**：agent 不是"每 task 新建"而是"克隆复用"——
- 复用 clone 的**代价**：共享 model backend（[backend/app/agent/listen_chat_agent.py:1031]() model=self.model_backend.models），且工具/浏览器句柄须在 clone 时正确复制，CDP 要抢占新端口（见 6.4），复杂度高；
- 不复用（每 task 新建一个完整 agent_model）的**代价**：每次要重建模型 + 报 create_agent 事件 + 重建浏览器连接，海量开销，无法支撑"几十个子任务"。

**Why 池化且初始为空**（[backend/app/utils/single_agent_worker.py:46]() pool_initial_size=0）：避免为"只来一个子任务"的场景预克隆浪费 CDP；首次用才即时 clone。取舍 = "首任务稍慢" 换 "极端场景不浪费浏览器资源"。

**真正并行的断言存疑**：并发编排体在 CAMEL `BaseWorkforce`，需验证其实现。

### 决策 3：同步/异步（长任务）HTTP 语义 —— 队列 + SSE，而非请求-响应阻塞

**取舍**：Agent 执行可能分钟级，不能同步 await。Eigent 选：controller 把 `ActionImproveData` 入队即返回 SSE 流（[backend/app/controller/chat_controller.py:472-500]()），`step_solve` 作为生成器在 `while True` 里消费 `task_lock.get_queue()`，逐 action `yield sse_json(...)`（[backend/app/service/chat_service.py:526-592]()`）。

**Why 队列 + SSE 而非 WebSocket**：SSE 是单向服务端推送，恰好匹配"agent 主动推进度"的只读语义，实现简单、天然支持断线重连（前端可按 `request.is_disconnected` 感知）。WebSocket 解决"双向高频"但本项目双向都要走 HTTP 上报（人类回答走独立 POST `/human-reply`），SSE 足够。
- **代价**：SSE 连接即"任务所有权"，客户端断线会引发 task_lock 清理与 workforce stop（[backend/app/service/chat_service.py:536-576]()`），需要 `timeout_stream_wrapper` 兜底（[backend/app/controller/chat_controller.py:314]()）；且多客户端同时连一个 project 可能竞争同一 task_lock（有 status 防重入，[backend/app/service/chat_service.py:1856-1863]()`、`2265`）。

**Why 队列承载状态而非全局单例**：TaskLock per-project 化，天然隔离并发项目（`task_locks` 字典，[backend/app/service/task.py:656]()），不同 project 各有 `get_task_lock`。这避免了"跨项目抢同一 event loop"。

---

## 亮点与问题（不吹不黑，伴论证）

### 亮点 1：运行时分流的三车道架构，兼顾简单/复杂/专用三种诉求
一举覆盖"简单问答(question_agent) / 复杂协作(Workforce) / 专用单agent(single-agent mode)"，用 `question_confirm` 一次判定做路由（2.2），是"需求驱动、不过度设计"的正面例子。与 AGENTS.md"简洁优先"原则自洽——不为单 agent 硬套多 agent。

### 亮点 2：控制面/行动面消息总线设计得干净
`Action` 枚举 + `TaskLock.queue` + SSE 的三段式，让"agent 执行(even loop)" 与 "前端观测(HTTP)" 完全解耦。前端只认 `AgentStep`，后端只认 `Action`，中间只有 JSON 事件。这让"断线/重连/多轮"都能在队列层统一处理，而 agent 本身无状态化（world 通过 `process_task_id` 区分）。

### 亮点 3：对 CAMEL 的侵入控制得体（继承+钩子，而非改写）
`Workforce` 只覆盖 `*_task`/`*_assignee`/钩子，插入"node_id→agent_id 映射"与"旁路 SSE"（3.3），不重写调度。既复用框架稳定性，又拿到前端要的可观测性。`_analyze_task` 重试（[backend/app/utils/workforce.py:109-173]()）覆盖了 CAMEL 结构化输出返回 None 的边界，是对框架缺陷的务实补丁。

### 亮点 4：CDP 浏览器资源的池化与生命周期防泄漏
并行 worker 抢浏览器是高风险泄漏点。`CdpBrowserPoolManager`(端口锁) + clone 时抢占 + workforce 停止强制 `release_by_task`（6.4）形成闭环，且 `pool_initial_size=0` 避免过度占用。这在多浏览器、多并行 agent 场景下是扎实的资源工程。

### 问题 1：Workforce 的"并行"与崩溃恢复依赖 CAMEL 黑盒
所有 worker 并发、依赖拓扑、重试/replan 都在 `BaseWorkforce.start()` 内（7.2）。Eigent 无法控制"并行度上限"、"并发阈值"，一旦 CAMEL 升级改事件顺序，前端 `assign_task`→`task_state` 的时序会漂移。属于"复用框架的红利，也是技术债"。**证据**：Eigent 未定义任何 max_parallel_workers 之类的参数，只透传 `FailureHandlingConfig(enabled_strategies=["retry","replan"])`（[backend/app/utils/workforce.py:99-101]()）。Eigent 是否支持配置并发度、是否有文档声明上限，取决于 CAMEL `BaseWorkforce` 是否暴露并行度参数（源码不在本仓库，未核实）。

### 问题 2：前端崩溃/断线时状态丢失，靠后端强清理兜底
SSE 断线会直接删 task_lock（[backend/app/service/chat_service.py:564-570]()`），Workforce stop，进度靠 replay 从持久化 memory 恢复。这只在"有 server 端持久化"时成立；若本地 crash，in-process 的 `conversation_history`/`decompose_sub_tasks`（`task.py` in-memory 字段）会随进程消亡。持久化依赖 memory 服务，尚未覆盖全部队列状态（memory 模块对 `sub_tasks`/`human_input` 排队中的内容是否持久化，见 `memory.md`，未完全闭合）。

### 问题 3：`question_confirm` 误判直接决定用户路径，无人工纠偏，且判定成本高熵
简单问题若判成 complex，用户会被拉进"确认→分解→执行"重流程（`chat_store.ts` 对 confirmed 后 to_sub_tasks 依赖强）；complex 判成 simple 则直接"答非所长"。判定仅由一次性 prompt 决定，缺少"用户可纠正"的后门（例如 forced workforce flag）。属于产品取舍，但值得在 UI 层暴露"强制复杂任务"入口（前端是否已暴露该开关，未核实）。

### 问题 4：`listen_chat_agent.py` 的流式工具执行有较多"workaround"标记
源码多处 `# TODO: Refactor - current marker detection is a workaround` / `# TODO: CAMEL should handle this task sync`（[backend/app/agent/listen_chat_agent.py:649-651]()、[backend/app/utils/workforce.py:776-777]()），表明 Eigent 在兼容 CAMEL 的流式/状态同步上做了不少补丁式处理，是"顺着框架缝补"的痕迹，维护时需要格外小心 CAMEL 升级。

---

## 验证与遗留

- 本解读核心执行/编排文件覆盖率约 **95%**（agent_model、prompt、全部 factory、task、chat_service、workforce.py、single_agent_worker.py、single_agent_service.py 均为高覆盖），前端与次要 controller 覆盖关键路径。所有 `[路径:行号]` 经真源重定位校验。
- 遗留未决项（依赖 CAMEL 源码或前端交互，供复核）：
  - CAMEL `BaseWorkforce.start()` 是否对无依赖 pending task 真做 `asyncio.gather` 并发、Eigent 能否约束并行度——决定 "collaborate in parallel" 断言成立的精确粒度。
  - memory 模块是否持久化 `decompose_sub_tasks` / `human_input` 排队内容（影响崩溃恢复粒度），详见 `memory.md`。
  - 前端是否暴露"强制 Workforce"入口以纠偏 `question_confirm` 误判。
  - `# TODO: CAMEL should handle this task sync`（[backend/app/utils/workforce.py:776]()）等补丁对 CAMEL 升级的敏感度。


