# OpenBee Agent Runtime 与 Go 并发模型

> 本解读按 agent 类规格产出，配套 [README.md](README.md)。本文深挖 OpenBee 的"运行时"层：AI 引擎的插件式接入、运行时可切换引擎、engine args 的层级合并，以及贯穿其中的 Go 并发惯用法（channel 信号量、单事件循环、context 取消）。

## 运行时架构：用插件抽象外部 AI 引擎

OpenBee 不内置 Agent 运行时，而是**把三个外部 CLI（Claude / Codex / Pi）当成可插拔引擎**，以自己的调度/执行循环驱动它们作为子进程。核心契约是 `EngineAdapter` 接口（[internal/ai/contracts.go:94-94]()）：

- `Prepare(workDir, opts)`：每次 Run 前的幂等初始化钩子（Claude 清理遗留配置文件，其余返回 nil）。
- `Run(ctx, workDir, prompt, opts, logPath)`：启动子进程执行，返回进程句柄、事件 channel、结果提取器。
- `CollectTokenUsage(ctx, sessionID)`：按引擎专属存储读每轮 token 用量。

<!-- snippet: type EngineAdapter interface { -->

引擎采用 **`Registry.Factory` 插件注册**：每个引擎在包 `init()` 里把自己的 `Factory` 注册进进程级 `DefaultRegistry`（[internal/ai/registry.go:42-42]()），按名构造。这让新增引擎只需注册一个 Factory，其余调度/执行逻辑零侵入。引擎常量在 `contracts.go` 的 `EngineClaude/Codex/Pi`（[internal/ai/contracts.go:47-51]()）。

<!-- snippet: func (r *Registry) Register(name string, f Factory) { -->

### RunResult：把"结果提取"绑定到当时选中的引擎

`EngineAdapter.Run` 返回 `RunResult`，其 `ExtractResult` 是一个**闭包，绑定在发起那次 Run 时实际选中的引擎上**（[internal/ai/contracts.go:77-77]()）。这是 OpenBee 与"引擎切换"正确性较量的关键：因为结果提取方式（从日志解析回复的规则）各引擎不同，如果异步等进程退出后再查"当前默认引擎"，可能因 `/engine` 切换而用错解析器。把 extractor 提前关闭在 Run 时刻，就保证 **引擎中途切换不影响在途执行的结果解析**。

<!-- snippet: type RunResult struct { -->

## 运行时可切换引擎：enginecfg + DynamicAdapter

"运行时给 Worker/Bee 换引擎"是 OpenBee 的差异化能力——`/engine` 命令可在运行时切换调度器（Bee）和 Worker 各自用的引擎。实现分两层：

### enginecfg.Store：并发安全的默认引擎名

`enginecfg.Store` 保存当前默认引擎名，用 `sync.RWMutex` 保护（[internal/domain/enginecfg/default.go:9-9]()），启动时从配置/DB 播种、注入到所有需要观察或切换默认引擎的子系统。`Resolve(workerEngine)` 方法实现"Worker 显式指定则优先，否则回退全局默认"（[internal/domain/enginecfg/default.go:34-38]()）。

<!-- snippet: type Store struct { -->
<!-- snippet: func (s *Store) Resolve(workerEngine string) string { -->

### DynamicAdapter：按调用时刻路由到当前引擎

单个 Worker 通常有一个显式 engine；但当引擎为空、要用运行时默认时，`DynamicAdapter` 包着所有已注册引擎，每次 `Run` 时通过 `cfg.Get()` 读当时的默认引擎再分发（[internal/ai/dynamic.go:36-36]()）。它的 `RunResult` 让 `ExtractResult` 关闭在**实际分发的那个引擎**上，所以异步处理结果不受后续 `/engine` 切换影响——这是"动态路由 + 结果绑定"配合的典范。

<!-- snippet: func (d *DynamicAdapter) Run(ctx context.Context, workDir, prompt string, opts RunOptions, logPath string) (RunResult, error) { -->

## Engine Args：三层合并的 CLI 参数注入

引擎要支持不同模型、思考深度等，靠 CLI 参数控制。OpenBee 用 **engine args** 分三层配置、按"全局 → 特定级"合并：

- **全局**：`SystemConfigKeyEngineArgsGlobal`（系统配置表存全局 engine args）
- **Worker 级**：`worker.EngineArgs`（JSON：`map[engine]rawCLIString`）
- **Bee 级**：`SystemConfigKeyEngineArgsBee`

`resolveEngineArgs` 把全局 map 与 worker map 用 `MergeEngineArgs` 合并，再取当前引擎那组（[internal/domain/worker/manager.go:117-117]()）。Bee 侧有独立的 `resolveEngineArgs`（[internal/domain/bee/bee_process.go:70-77]()），合并全局 + bee 级。

<!-- snippet: func (m *Manager) resolveEngineArgs(ctx context.Context, worker model.Worker, engineName string) []string { -->

### MergeEngineArgs：后置覆盖前置

`MergeEngineArgs` 把 override 追加在 base 之后（[internal/ai/engine_args.go:106-106]()）。CLI 惯例是**后出现的 flag 覆盖前者**，所以 worker 参数追加在全局参数后，worker 可覆盖全局的同一 flag，无需"冲突检测"。这是用拷贝切片做浅合并、保持原始顺序的轻量实现。

<!-- snippet: func MergeEngineArgs(base, override EngineArgsMap) EngineArgsMap { -->

### 自写 tokenizer 规避 shell 注入

engine args 是**用户输入的原始 CLI 字符串**（如线程数、目录、model 参数），要作为子进程 argv 传给引擎，绝不能经 shell 展开。OpenBee 自写 `splitCLIArgs` 逐字符状态机解析引号/转义/空白（[internal/ai/engine_args.go:27-27]()），产出安全的 argv 切片，而不是 `sh -c`。这是"把不可信用户输入当代码参数而非 shell 文本处理"的正确姿势。

<!-- snippet: func splitCLIArgs(s string) ([]string, error) { -->

## 执行链路：从任务到 Worker 子进程

`ExecuteWorker` → `launchRuntime` → `engine.Run` 是 Worker 执行的核心（[internal/domain/worker/execution.go:29-29]()）。`launchRuntime` 做了三件与正确性相关的事：

1. **生成 worker 专属 RPC token**：`GenerateWorkerToken` 用 `permission_scopes`（SplitAndTrim）+ TTL 生成 JWT，作为 `APIKey` 注入引擎环境（[internal/domain/worker/execution.go:79-82]()）。Worker 只能用这个 scope 受限的 token 调用 RPC，不能越权。
2. **context 取消传播**：`execCtx` 继承 dispatcher 的 ctx，加超时/取消（[internal/domain/worker/execution.go:86-92]()）。这样任务取消、`/clear`、shutdown 的取消信号能一直传导到 `engine.Run` 的子进程——**取消是沿 context 树传播的**。
3. **extra env + extra args 注入**：`ResolveWorkerEnv` 解密的加密环境变量 + 合并后的 engine args 一起进 `RunOptions`（[internal/domain/worker/execution.go:29-29]()）。

<!-- snippet: func (m *Manager) ExecuteWorker(ctx context.Context, req ExecuteRequest) (model.WorkerExecution, error) { -->

### monitorExecution：事件循环收尾 + 兜底

`engine.Run` 返回后，后台 goroutine 遍历 `runRes.Output` channel，按 `done`/`error` 事件更新 execution/worker 状态。**兜底逻辑**：如果 Output channel 关闭却没收到终止信号（进程被杀/崩溃/信号终止），就调用 `MarkAbandoned` 把 execution 标记为 failed，防止"僵尸 running"记录（[internal/domain/worker/execution.go:123-123]()）。这呼应 SQLite 单连接下 execution 状态机的一致性诉求。

<!-- snippet: func (m *Manager) monitorExecution(exec model.WorkerExecution, worker model.Worker, runRes ai.RunResult, cancel context.CancelFunc, logPath string) { -->

## Go 并发惯用法：三个可迁移的模式

OpenBee 的并发代码不炫技，但每个地方都选对了原语，值得单拎出来：

### 1. channel 信号量：Feeder 限制并发 Bee 进程

Feeder 用带缓冲 channel 当信号量限制同时在跑的 Bee 进程数：`sem = make(chan struct{}, cfg.Feeder.MaxConcurrentBee)`（[internal/domain/bee/feeder.go:62-62]()）。每个 tick 先数"可用槽位" `available := cap(f.sem) - len(f.sem)` 再 `ClaimBatch(available)`（[feeder.go:141-143]），确保"每批 claim 的消息都能立刻占用槽位启动"，不因半满而浪费。这是 **Go 官方的"buffered channel as counting semaphore"** 惯用法：无需显式锁，容量即并发上限。

<!-- snippet: 	sem             chan struct{} // bounds concurrent bee processes -->
<!-- snippet: 	available := cap(f.sem) - len(f.sem) -->

### 2. 单事件循环 + 按 key 队列：Dispatcher 串行化每个 Worker

Dispatcher 是**单个 goroutine 跑一个 `select` 事件循环**，处理 `inCh`（新任务）/`resultsCh`（执行结果）/`clearCh`/`cancelCh`/`ctx.Done()`（[internal/domain/task/dispatcher.go:141-141]()）。每个 Worker 一个 `queueState{executing, pendingTasks}`：新任务落到某个 worker 时，若在跑则 append 到该 worker 的 pending 队尾，否则 `startTask`（[dispatcher.go:162-176]）。这保证了**同一 Worker 的任务天然串行**，且因为所有状态都由单循环串行触碰，map 无需加锁——彻底规避了"每 worker 一个 goroutine + 共享 map 锁"的复杂度。

<!-- snippet: func (d *TaskDispatcher) Run(ctx context.Context) { -->
<!-- snippet: func (d *TaskDispatcher) handleInbound(task DispatchTask) { -->

### 3. context 树传播取消：从用户操作到子进程

从 dispatcher 的 `context.WithCancel(d.ctx)` 开始（[internal/domain/task/dispatcher.go:178-178]()），到 launchRuntime 的 `context.WithTimeout/WithCancel`，再到 `engine.Run` 收到 `execCtx`——**取消信号沿 context 树层层传导，最终杀死 Worker 子进程**。OpenBee 用 context 不只传 deadline 值，而是承载"取消语义"贯穿整个运行时，这是 Go 组织跨层取消的标准范式。

<!-- snippet: func (d *TaskDispatcher) startTask(key string, task DispatchTask) { -->

## 全局关联

Runtime 层把三个异构外部引擎收敛成统一 `EngineAdapter`，是 [README.md](README.md) 架构图的"AI Engine Adapters"那一格的实体。它与 [data-model.md](data-model.md) 的 `bee_executions`（记录每次运行用了哪个引擎、状态、PID）和 `bee_session_contexts`（按 engine 键控跨消息续上下文）直接耦合；engine args 的全局配置存于系统配置表，属于数据层的 `bee_system_configs`。整套"多引擎 + 运行时切换"设计，是 [rbac.md](rbac.md) 的多用户维度之外的另一个能力支点——它让同一个 Worker 可以在不重建仓库的情况下切换思考引擎。
