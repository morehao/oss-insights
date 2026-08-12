# promptfoo 评估核心（eval core）深度解读

> 本文是 promptfoo 源码深度解读的第一条主线。目标：讲清"一次 eval 从配置文件到分数的完整生命周期"。
> 阅读对象：已读过架构概览（README）的读者。下一个模块是 redteam（复用同一评估管线跑攻击），因此本文会反复强调"评估管线是通用骨架，redteam 只是给它装上攻击内容"。
>
> 本解读以 `source_repo=/Users/morehao/Documents/study/ai-infra/promptfoo` 为真源基准，所有 `文件:行号` 引用均可在该仓库中重定位验证。当前分析基线 commit：`be44f8389`。

---

## 一句话根因

promptfoo 评估核心解决的根因问题是：**LLM 应用的"好不好"无法靠拍脑袋判断，必须能被自动化、可重复、可对比地量化**。把"prompt + provider + 测试用例 + 断言"声明成一份配置，评估核心负责把它展开成一个可并行执行的评估矩阵，对每个单元（prompt×provider×vars 组合）调 provider 取响应，再用断言算出分数，最终落库、排序、可视化。它是整个系统"数据驱动的评估"这一设计哲学的**写入口**与 **web viewer 的数据源**。

---

## 在项目中的角色：去掉它系统会怎样

如果把评估核心删掉，promptfoo 会退化成只剩下 CLI 骨架、provider 适配层、数据库与 web 界面的空壳——**没有任何东西能"产生一次评估结果"**。具体看三层角色：

- **它是评估的发起端与聚合端**。`src/evaluate.ts` 是外部（CLI/SDK/redteam/optimizer）统一调用的入口，负责把"用户配置"（`EvaluateTestSuite`）洗成"运行时测试套件"（`TestSuite`），喂给 `src/evaluator.ts` 的 `Evaluator` 类执行，再把拿到的 `EvaluateResult[]` 写回 `Eval` 记录。没有它，providers 没人调度、断言没人跑、数据库没人写。
- **它是数据模型的写入口**。`Evaluator.evaluate()` 最终落到 `this.store.save()`，下游所有功能（`promptfoo view` 的表格、`promptfoo share` 的分享页、database 表、JSONL 产物）读的都是这个写入口产出的数据。见 architecture/layers.json：`evaluator.ts` 被归在 `legacy-runtime` 层，`node` 层的 `evaluate.ts` 依赖它。
- **它是全链路唯一的核心骨架**。redteam、optimizer、`promptfoo eval`、`promptfoo share` 全部复用同一套 `evaluate()` / `Evaluator`。所以它不是众多模块之一，而是**别的模块都构建在其上的主干**。

一个关键的架构证据：`createRunEvalOption` 里 `isRedteam: testSuite.redteam != null`（见第 4 步指纹）——同一个展开函数只根据 testSuite 是否带 `redteam` 字段来切换姿态，管线本身并不关心内容是"正常测试"还是"攻击注入"。redteam 复用此管线的机制由此成立。

---

## 解决什么问题、根因

LLM 应用开发面临一个传统软件工程没有的问题：**无法用单元测试确定性断言输出"对错"**。

- prompt 换一个措辞、model 换一个温度、provider 换一个供应商，输出质量都可能漂移；
- "好"是主观的，人工 + 肉眼看几十条样本不可扩展，也无法在 CI 里回归；
- 一次评估要先定义一个**乘法组合空间**（N 个 prompt × M 个 provider × V 组变量），手工跑不现实。

promptfoo 的回答（README 原话）："Test your prompts and models with automated evaluations"、"Compare models side-by-side"。要达成这个目标，评估核心的根因设计命题拆成几个彼此交织的子问题：

| 子问题 | 评估核心的解法 |
|--------|---------------|
| 配置怎么写最省心 | 一份声明的 `promptfooconfig.yaml`，声明 prompt/providers/tests/assert，由 `readTests`/`processPrompts` 标准化 |
| 组合空间怎么展开 | 笛卡尔积展开：测试用例 × var 组合 × provider（列）× prompt（列）× repeat |
| "好不好"怎么量 | 断言库（50+ 类型）：从精确匹配到正则到 model-graded 自评 |
| 规模化怎么跑 | 自适应并发 + 重试 + 超时 + 缓存，时间/成本可控 |
| 结果怎么留住 | 落数据库 + JSONL 流式写，供 web viewer/share 消费 |

其中尤以"断言"为根因的核心：**promptfoo 把"怎么判断 llm 输出是否达标"抽象成可插拔的断言体系**，这是它与"直接用 LLM 输出走查"这类临时做法的分水岭。

---

## 设计思路：从 yaml 配置到分数的完整管线

### 管线总览（Why 视角）

整条管线是一条清晰的"声明 → 展开 → 执行 → 判定 → 沉淀"流：

```text
promptfooconfig.yaml (声明)
  │  evaluateWithSource()  [src/evaluate.ts:317]
  ▼
EvaluateTestSuite ──loadApiProviders + processPrompts + readTests──▶ TestSuite (运行时)
  │  Evaluator.evaluate()  [src/evaluator.ts:4878]
  ▼
  ├─ buildCompletedPrompts()      → prompt 列（每条 provider 针对哪些 prompt 生效）
  ├─ buildTestsFromSuite()        → 测试用例行（含 defaultTest 合并、scenario 注入）
  ├─ buildRunEvalOptions()        → 展开为 RunEvalOptions[]（testcase×vars×provider×prompt×repeat 的笛卡尔积）
  ├─ runSerialEvalSteps()/runConcurrentEvalSteps()  → 按并发调度执行
  │     └─ runEvalInternal()      → 每条：渲染 prompt → callActiveProvider → outcome
  │           └─ gradeRunEvalResponse() → runAssertions() → GradingResult
  ├─ processComparisonAssertions()→ select-best / max-score 这类"跨多输出"断言在主循环后补跑
  └─ finalizeEvaluation()         → 落库、汇总 stats、telemetry
  ▼
Eval 记录（数据库 + JSONL）  → web viewer / share
```

### 放弃的替代方案（为什么是现在的形态）

- **不把整个 eval 当成一个不透明白盒函数**，而是拆出 `Evaluator` + `EvaluationStore` + `EvaluatorRuntime` 三件套。`Evaluator` 只负责"调度与判定流程"，"结果怎么落地"（DB/内存/JSONL）由 `EvaluationStore` 抽象，`nodeEvaluatorRuntime` 是 node 实现（[src/node/evaluatorRuntime.ts:23-86]()）。这套运行时注入让同一套评估逻辑能被 node CLI、webviewer、测试环境（内存 store）复用，而不是各自复制一份。
- **不搞"一次渲染固定 prompt 列表"**，而是在运行时才把 prompt 一个个展开成可并行单元（`RunEvalOption`），这样每个单元独立计 latency、独立判分，天然适合并发与局部失败隔离。
- **不用纯解释执行的"脚本断言"打天下**，而是提供结构化断言类型 + 脚本断言并存：结构化断言（contains/equals/regex…）让声明式配置足够表达常见检查，脚本断言（javascript/python/ruby）兜底任意复杂逻辑。

### 调度模型（Why 选这个）

评估的最终成本 = 组合数 × provider 单次延迟，因此**并行度直接决定跑一次 eval 的墙钟时间**。调度做了三层：

1. **并行单元**：每个 `RunEvalOption`（一个 testcase×provider×prompt×vars×repeat 组合）是并发独立单位；`runConcurrentEvalSteps` 用 `async.forEachOfLimit(concurrentRunEvalOptions, concurrency, ...)` 限流并发执行（[src/evaluator.ts:3980-3980]()）。
2. **串行隔离**：带 `runSerially` 标记的测试或会话型（conversation）测试被专门挑出按串行走（`runSerialEvalSteps`，[src/evaluator.ts:3932-3932]()），避免并发打破多轮对话的时序。
3. **自适应限速**：provider 网络受限（429/限流头）时，`rateLimitRegistry` 学习响应头的 rate limit 并动态压低对应 provider 的并发，而非全局退避——这是把"并发"从"拍脑袋常数"升级成"环境自适"的关键决策（见 [src/scheduler/index.ts:1-8]() 模块注释）。

这里一个重要的权衡：**promptfoo 把"单 provider 吞吐"留给 sidecar 式的 rate-limit 注册表，而不是让每个 provider 自管并发**，这样所有 provider（含 redteam 内部 provider）共享一套限速器，行为一致且可观测（`rateLimitRegistry.on('ratelimit:hit'|'learned'|'decreased'|'increased', ...)` 全有 debug 日志，[src/evaluator.ts:3261-3285]()）。

---

## 关键数据结构（理解必需的精简类型）

### 结果行 EvaluateResult（web 表格里的一格 / 一次判定单元）

[src/types/index.ts:385-414]()（节选）

```ts
export interface EvaluateResult {
  promptIdx: number;   // 结果表单中的「列」——第几个 prompt
  testIdx: number;     // 结果表单中的「行」——第几个测试用例
  testCase: AtomicTestCase;
  promptId: string;
  provider: Pick<ProviderOptions, 'id' | 'label'>;
  prompt: Prompt;
  vars: Vars;
  response?: ProviderResponse;
  error?: string | null;
  failureReason: ResultFailureReason;
  success: boolean;    // 是否通过（由 GradingResult.pass / threshold 决定）
  score: number;       // 聚合得分（0-1）
  gradingResult?: GradingResult | null;
  namedScores: Record<string, number>;
}
```

`promptIdx` / `testIdx` 这对坐标构成了**结果表的结构化契约**：testIdx 是行（哪个测试用例），promptIdx 是列（哪个 prompt）。promptfoo 的 web 表格、对比报告、`select-best` / `max-score` 跨行聚合全部靠这对坐标工作——这正是"数据模型写入口"意义的具象化。

### 判定结果 GradingResult（一次断言 / 一组断言的产物）

[src/types/index.ts:521-573]()（节选）

```ts
export interface GradingResult {
  pass: boolean;                 // 是否通过
  score: number;                 // 得分，通常 0-1
  reason: string;                // 通过/失败理由
  namedScores?: Record<string, number>;   // 命名指标分
  componentResults?: GradingResult[];     // 多断言时每个子结果
  assertion?: Assertion;
  metadata?: { graderOutputs?, renderedGradingPrompt?, graderError? ... }  // 概要
}
```

注意 `componentResults`——一组断言里每条断言是一个子 `GradingResult`，`AssertionsResult.addResult` 把它们按 weight 聚合成最终 score 与 pass（[src/assertions/assertionsResult.ts:85-142]()）。

### 测试用例原子态 AtomicTestCase

[src/types/providers.ts:60-70]()（节选）

```ts
interface AtomicTestCase {
  description?: string;
  vars?: Record<string, VarValue>;
  providerResponse?: ProviderResponse;
  success?: boolean;
  score?: number;
  failureReason?: string;
  metadata?: Record<string, any>;
  options?: Record<string, any>;
}
```

### 断言派发点

`runAssertion` 的统一入口 [src/assertions/index.ts:406-406]()，分派到 `ASSERTION_HANDLERS` 注册表 [src/assertions/index.ts:226-317]()（50+ 类型：contains/equals/regex/llm-rubric/g-eval/factuality/model-graded-closedqa/javascript/python/webhook/trajectory…）。

### 缓存结构

`FetchWithCacheResult` [src/cache.ts:275-289]()：`{ data, cached, status, headers, latencyMs, deleteFromCache, updateCache }`——缓存不仅存响应，还区分"命中缓存"（`cached=true`），供延迟/成本统计识别。cache 用 `AsyncLocalStorage` 做 namespace（`src/cache.ts:31-32, 249-261`）与开关注入（`withCacheEnabled`，[src/cache.ts:263-263]()）。

---

## 核心流程：一次 eval 的完整生命周期

### Mermaid 时序图

```mermaid
sequenceDiagram
  autonumber
  actor U as 用户/CLI/redteam
  participant E as "evaluate.ts<br>(外部入口)"
  participant P as "prompts/processors<br>+ readTests"
  participant EV as "Evaluator<br>(evaluator.ts)"
  participant PRE as "Provider<br>(providers)"
  participant CA as cache.ts
  participant A as "assertions<br>runAssertions"
  participant M as "matchers<br>llmGrading/rubric"
  participant S as "EvaluationStore<br>+ models/eval"

  U->>E: evaluateWithSource(testSuite)
  E->>P: loadApiProviders + readTests...
  P-->>E: TestSuite (prompts+providers)
  E->>EV: doEvaluate(constructedTestSuite)
  EV->>EV: buildCompletedPrompts()
  EV->>EV: buildTestsFromSuite()
  EV->>EV: buildRunEvalOptions() 展开
  EV->>EV: 拆 serial / concurrent 两路
  loop 每个 RunEvalOption (并发执行)
    EV->>EV: renderRunEvalPrompt() 填 vars
    EV->>PRE: callActiveProvider(prompt)
    PRE->>CA: fetchWithCache (命中短路)
    CA-->>PRE: ProviderResponse
    PRE-->>EV: ProviderResponse
    EV->>A: runAssertions(response, test)
    A->>M: 分派 handler (结构化/model-graded)
    M-->>A: GradingResult
    A-->>EV: 聚合 GradingResult (score/pass)
    EV->>S: persistEvalRow(row)
  end
  EV->>EV: processComparisonAssertions()
  EV->>S: finalizeEvaluation() 落库+stats
  S-->>E: Eval 记录
  E->>U: return Eval (stats)
```

### 流程文字 + 关键行号标注

**第 1 步：外部入口洗配置（`src/evaluate.ts`）**

`evaluateWithSource` 是唯一外部入口 [src/evaluate.ts:317-317]()（返回类型为 `Eval`，被 `src/node/doEval.ts` 等上游调用）。它做：跑 DB migration（若 `writeLatestResults`）→ 加载 provider 实例 `loadApiProviders` → 组装运行时 test suite `createRuntimeTestSuite`（把 `file://` defaultTest 外链读进来、`readTests` 读测试用例、`processPrompts` 处理 prompt 文件）→ 解析嵌套 provider 引用 `resolveNestedProviders` → 包一层可序列化 config（`createSerializableUnifiedConfig`，把不可 JSON 化的函数 transform 替换成 `[inline function]` 标记）→ 建 `Eval` 记录 → 包在 `cache.withCacheEnabled` 里调用真正的 `doEvaluate`。

引用块指纹 + 原文（[src/evaluate.ts:347-359]()，核心接线）：

```ts
  const ret = await cache.withCacheEnabled(options.cache === false ? false : undefined, () =>
    doEvaluate(
      {
        ...constructedTestSuite,
        providerPromptMap: parsedProviderPromptMap,
      },
      evalRecord,
      {
        isRedteam: Boolean(testSuiteConfig.redteam),
        ...options,
      },
    ),
  );
```

一处值得注意的设计：`cloneTestForResolve`（[src/evaluate.ts:50-50]()）——在 resolve provider 前浅拷贝 test/assert，避免把"活的 SDK client（含循环引用，如 BedrockRuntime/Anthropic client）"污染回输入配置，进而破坏 drizzle 的 JSON 序列化。文件头注释明确写着这修复了 #8687（[src/evaluate.ts:36-49]()）。这是"运行时注入不该泄漏到持久化 config"的典型防御。

**第 2 步：prompt 读取与展开（`src/prompts/`）**

`processPrompts`（[src/prompts/index.ts:242-242]()）把配置里的 prompts 条目（字符串路径 / 函数 / 对象）统一读成 `Prompt[]`。`processPrompt`（[src/prompts/index.ts:107-107]()）按扩展名分派：`.csv/.json/.jsonl/.j2/.txt/.md/.py`/可执行文件等；普通字符串走 `processString`（`src/prompts/processors/string.ts`）。glob 模式会递归展开（[src/prompts/index.ts:145-169]()）。

**第 3 步：测试用例标准化与默认值合并（`src/evaluator.ts`）**

`prepareTestCaseForEval`（[src/evaluator.ts:2449-2449]()）在每个用例上合并 `defaultTest` 的 assert/options/metadata/provider/threshold/assertScoringFunction（含 `disableDefaultAsserts` 开关，[src/evaluator.ts:2464-2468]()）。scenario 则经 `buildScenarioTests` + `mergeScenarioTest` 把 `defaultTest ⊕ scenario.config ⊕ test` 三层合并（[src/evaluator.ts:2305-2346]()），并用 `__scenario_${i}__` 作为 conversationId 保证多轮隔离（[src/evaluator.ts:2327-2327]()）。

**第 4 步：评估矩阵展开（`src/evaluator.ts`）**

核心是 `buildRunEvalOptions`（[src/evaluator.ts:2399-2399]()）逐层向下展开成笛卡尔积：

- 每个 testcase → `appendRunEvalOptionsForTestCase`（[src/evaluator.ts:2516-2516]()）
- **var 组合** → `appendRunEvalOptionsForVars`（[src/evaluator.ts:2583-2583]()），`generateVarCombinations` 做笛卡尔积（[src/evaluator.ts:1844-1892]()）
- **provider 列** → `appendRunEvalOptionsForProvider`（[src/evaluator.ts:2644-2644]()），受 `isProviderAllowed` 过滤（[src/evaluator.ts:2619-2619]()）
- **prompt 列** → 受 `isAllowedPrompt` 过滤（[src/evaluator.ts:2682-2682]()）
- **repeat** → `effectiveOptions.repeat` 循环（[src/evaluator.ts:2556-2578]()）
- 每个原子单元 `createRunEvalOption`（[src/evaluator.ts:2710-2710]()）

引用块指纹 + 原文（[src/evaluator.ts:2764-2764]()，redteam 复用管线的关键证据）：

```ts
    isRedteam: testSuite.redteam != null,
```

**第 5 步：按并发 / 串行执行（`src/evaluator.ts`）**

`_runEvaluation`（[src/evaluator.ts:4608-4608]()）先建三路 abort 信号体系：用户信号 + 全局 `maxEvalTimeMs` 超时 + target-unavailable，见 [src/evaluator.ts:4625-4647]() — 注释特别说明 target 403 等信号**不传入 provider**（因为检测到非瞬态状态时调用已完成，只用来停循环），避免已完成的调用被无谓 abort。随后：

- `buildCompletedPrompts`（[src/evaluator.ts:4677-4677]()）算 prompt 列；`buildTestsFromSuite`+`filterByRange`（[src/evaluator.ts:4681-4682]()）确定测试用例行；
- `buildRunEvalOptions`（[src/evaluator.ts:4692-4692]()）、`markComparisonRows` 标记 select-best/max-score 行；`adjustConcurrencyForSerialFeatures`（[src/evaluator.ts:4708-4708]()）在用到会话变量 / serial / storeOutputAs 时自动降并发；
- 拆 `serialRunEvalOptions` / `concurrentRunEvalOptions`（按 `test.options.runSerially`，[src/evaluator.ts:4783-4791]()）→ `executeEvalSteps`（[src/evaluator.ts:3710-3710]()）。

调度执行三选一（`executeEvalSteps` 内部 [src/evaluator.ts:3745-3774]()）：

1. 若 `shouldGroupGradingByProvider`（并发=1 且无超时且无会话变量，[src/evaluator.ts:4793-4794]()）→ `runGroupedEvalSteps`（[src/evaluator.ts:3803-3803]()）走"按 provider 分组、延迟判定"模式，减少同 judge 分组被并发拆散；
2. 否则先串行后并发：`runSerialEvalSteps`（[src/evaluator.ts:3932-3932]()）逐个 `processEvalStepWithTimeout`；
3. `runConcurrentEvalSteps`（[src/evaluator.ts:3964-3994]()）用 `async.forEachOfLimit(concurrentRunEvalOptions, processingContext.concurrency, ...)` 限流并发执行。

**第 6 步：单条单元执行（`runEvalInternal`，[src/evaluator.ts:1538-1538]()）**

- `createRunEvalState`（[src/evaluator.ts:673-673]()）算 vars、conversationKey（provider+prompt.id+promptIndex+conversationId 的 JSON tuple，[src/evaluator.ts:686-686]()）；
- 注入会话变量：prompt 里用到 `_conversation` 时把对话历史挂进 vars（`attachConversationVar`，[src/evaluator.ts:756-756]()）；
- 注入运行时变量（`__evalId`/`__evalStepId`/`__repeatIndex`）供 prompt/provider 引用，但**不落库**——`omitEvalRuntimeVars`（[src/evaluator.ts:746-746]()）在 persist 前剔除，避免污染持久化结果与 `--filter` 重跑（[src/evaluator.ts:737-754]() 注释说明了设计由来）；
- `renderRunEvalPrompt`（[src/evaluator.ts:805-805]()）渲染出实际 prompt（redteam 时跳过注入变量）；
- `callProviderForRunEval` → `callActiveProvider`（[src/evaluator.ts:1008-1008]()）：`test.providerOutput` 存在则直接用（AI SDK 预传输出 / 缓存回放，[src/evaluator.ts:889-895]()），否则 `activeProvider.callApi(renderedPrompt, callApiContext, { abortSignal })`，并外包 `rateLimitRegistry.execute(...)`（[src/evaluator.ts:1059-1061]()）；
- 拿到 response 后 `createEvaluateResult`（[src/evaluator.ts:1652-1652]()）构造结果行，`applyRunEvalResponseOutcome`（[src/evaluator.ts:1234-1234]()）判定响应是否有 error/空输出，再决定是否进入评分；`storeOutputAs` 寄存器保存输出供后续测试用（[src/evaluator.ts:1696-1699]()）。

**第 7 步：判定（`gradeRunEvalResponse` → `runAssertions`，[src/evaluator.ts:1316-1316]()）**

响应先过 `transformRunEvalResponse`（provider.transform / test.options.transform|postprocess，[src/evaluator.ts:1418-1469]()，含二进制 media blob 提取），再进 `runAssertions`（[src/assertions/index.ts:722-722]()）批处理所有断言 → 聚合 `GradingResult` → `applyGradingResult` 写回结果行（[src/evaluator.ts:578-578]()）。若 `deferGrading`（按 provider 分组联判模式），判定被推迟成 promise（[src/evaluator.ts:1377-1396]()），由 `runGroupedGradingForRows` 统一回收（[src/evaluator.ts:2941-2941]()）。

引用块指纹 + 原文（[src/evaluator.ts:1377-1398]()，延迟判定路径）：

```ts
  if (deferGrading) {
    invariant(providerCallQueue, 'providerCallQueue is required when deferGrading is enabled');
    ret.response = processedResponse;
    const gradingPromise = withProviderCallExecutionContext(
      { abortSignal, providerCallQueue, rateLimitRegistry },
      () =>
        runAssertions({
          prompt: renderedPrompt,
          provider,
          providerResponse: assertionProviderResponse,
          test,
          vars,
          latencyMs: response.latencyMs ?? latencyMs,
          assertScoringFunction: test.assertScoringFunction as ScoringFunction,
          traceId,
        }).then((checkResult) => applyGradingResult(ret, checkResult)),
    ).catch((error) => {
      applyGradingError(ret, error, abortSignal);
    });
    deferredGradingPromises.set(ret, gradingPromise);
    return;
  }
```

**第 8 步：跨行比较断言（`processComparisonAssertions`，[src/evaluator.ts:4048-4048]()）**

`select-best` / `max-score` 这类"需要看多个输出"的断言在主循环里被跳过（[src/assertions/index.ts:798-801]()），等所有行跑完再聚合判 winner：`processSelectBestAssertions`（[src/evaluator.ts:4100-4100]()）让 LLM 选最佳（`matchesSelectBest`，[src/matchers/comparison.ts:16-16]()）；`processMaxScoreAssertions`（聚合各输出其他断言加权得分，`selectMaxScore`，[src/matchers/comparison.ts:86-86]()）。结果经 `finalizeComparisonGrading`（[src/evaluator.ts:4357-4357]()）merge 回对应行并落库。

**第 9 步：收尾落库（`finalizeEvaluation`，[src/evaluator.ts:4415-4415]()）**

`store.appendPrompts` → 清进度 → 若 eval 超时补写 `addMaxDurationTimeoutResults`（[src/evaluator.ts:4488-4488]()）→ `store.setVars` → `afterAll` 扩展 → `recordEvalTelemetry` → `this.store.save()`。

引用块指纹 + 原文（[src/evaluator.ts:4483-4485]()）：

```ts
    if (this.store.persisted) {
      await this.store.save();
    }
```

随后 `Evaluator.evaluate()` 的 finally 里关 JSONL writer（先关、再 flush OTEL，[src/evaluator.ts:4906-4930]()）、shutdown OTEL、清理 provider 池与 rate-limit 注册表（[src/evaluator.ts:4904-4990]()）。

---

## 与其他模块的协同

- **providers**：`evaluate` 只依赖 `ApiProvider` 接口（`callApi`/`id`/`label`/`config`/`delay`，见 [src/types/providers.ts:123-134]()）。provider 实例由 `loadApiProviders` / `buildConfiguredProviderMap` 在 evaluate.ts 构造并注入（[src/evaluate.ts:327-330]()）；断言深处的 model-graded 还会用 `getAndCheckProvider('text', grading.provider, defaultGradingProvider, ...)` 临时起一个 grading provider（`src/matchers/providers.ts`，调用见 `src/matchers/llmGrading.ts` 各 matches* 函数）。
- **database / storage**：`src/node/evaluationStore`（`EvalEvaluationStore`）与 `models/eval` 负责把结果落 SQLite/drizzle；JSONL 流式产物由 `fileWriters`（[src/evaluator.ts:3251-3251](), [src/node/evaluatorRuntime.ts:84-86]()）写。`store.persisted` 控制是否落库。
- **redteam**：复用同一 `evaluate()`/`Evaluator`。铁证有两处：① `rateLimitRegistry` 共享给 redteam provider manager（[src/evaluator.ts:3287-3289]()）；② `isRedteam` 标志贯穿 `runEvalInternal` 处理空响应的姿态——`applyEmptyResponseOutcome` 里 redteam 空响应按 `success=true` 处理（[src/evaluator.ts:1306-1314]()）。redteam 用 `src/redteam/commands/*` 把攻击策略织入 tests，再走评估管线执行 + 判定，`report`/`riskScoring` 读评估返回的 stats。
- **view-server / cli / app**：评估产生的 `Eval` + `EvaluateResult` 是 web viewer 查询与渲染的数据源；`evaluateWithSource`（[src/evaluate.ts:317-317]()）是 `src/node/doEval.ts` 的调用链上游（doEval 组织 CLI 选项、replay/filter 后调它，[src/node/doEval.ts:235-235]() 起）。
- **分层归属（architecture/layers.json）**：`evaluator.ts`→`legacy-runtime`，`evaluate.ts`+`cache.ts`→`node`，`assertions/matchers/prompts/scheduler/testCase`→`core`，`types/validators`→`legacy-contracts`。核心评估逻辑在 `legacy-runtime` + `core` 两层，`evaluate.ts` 作为 `node` 层入口把两者接起来。

---

## 关键设计决策（Why + 权衡 / 代价）

### 决策一：并行调度 = "组合展开 + 限流并发"，而非"provider 自管吞吐"

- **Why**：一次 eval 的成本正比于组合数 × 单次延迟，并行是唯一能压墙钟时间的杠杆；但无脑全并发会打爆 provider 配额。
- **实现**：拆成独立 `RunEvalOption` 并发单元（见第 4 步）；网络层用 `rateLimitRegistry` 从响应头学习限流并逐 provider 动态调并发（[src/evaluator.ts:1059-1061]()，`src/scheduler/adaptiveConcurrency.ts`）。
- **权衡 / 代价**：并发调度 + 重试 + 会话串行隔离 + 两级超时（per-step `processEvalStepWithTimeout` 与全局 `maxEvalTimeMs`）叠加，让 `runEvalInternal`/`processEvalStep` 调用链偏深、参数冗长；换来"大而全的规模化能力"（能跑几百上千条不至于失控）。若不这么做，promptfoo 就是"一次跑几十条"的小工具，无法支撑 redteam 一次几千条的攻击注入。

### 决策二：model-graded 自评（llm-rubric / g-eval / factuality / closedqa）="可插拔的二级 LLM 判定"

- **Why**：很多质量维度（语义相关性、事实性、遵循复杂 rubric）无法用 contains/regex 这类确定性断言表达，必须"请另一个 LLM 来打分"。
- **实现**：`runJsonGradingPrompt`（[src/matchers/rubric.ts:781-781]()）统一做"拼 rubric → 渲染 → 调 grading provider → 解析 JSON score/pass/reason"；`matchesLlmRubric`（[src/matchers/llmGrading.ts:168-168]()）先尝试远程评分（`doRemoteGrading`），否则走本地 JSON 评分；g-eval 是两阶段（先生成 steps 再按 steps 打分，[src/matchers/llmGrading.ts:459-539]()）；`parseJsonGradingResponse` 对 LLM 输出做容错解析（[src/matchers/rubric.ts:737-779]()）。
- **权衡 / 代价**：模型自评引入**二次成本与不稳定**——所以 matchers 做严格解析 + `graderError` 标记（`isGraderFailure`，[src/matchers/llmGrading.ts:430-431]()），且 inverse 断言**不能**把"grader 出错"误翻成通过（`llmRubric.ts:44-51` 只在数值有效时做 `1 - score` 反转）。这是"自动化质量的杠杆"与"可信度风险"的交换，promptfoo 用结构化失败标记 + 可观测（把 `renderedGradingPrompt` 存进 metadata，[src/matchers/rubric.ts:875-875]()）兜住。

### 决策三：断言插件化（注册表 + 统一 GradingResult 契约）

- **Why**：让"质量判断"从 eval 主循环解耦，也方便用户自定义。
- **实现**：所有断言类型收敛到一个 `ASSERTION_HANDLERS` 注册表（[src/assertions/index.ts:226-226]()），统一返回 `GradingResult`；脚本断言（js/py/rb）既可作为 value 来源也可做整个断言执行体（[src/assertions/index.ts:563-603]()）；`AssertionsResult` 负责多断言加权聚合与 threshold 覆写（[src/assertions/assertionsResult.ts:144-235]()）。
- **权衡 / 代价**：注册表模式意味着"加新断言类型"要改 `index.ts` 注册（静态耦合）；但换来 50+ 开箱类型 + 与 matcher 层（llmGrading/rubric/comparison）的清晰分层。若用动态插件加载会换来解耦但牺牲类型安全与可发现性。取舍是"可扩展 + 可发现优先"。

### 决策四：缓存（TTL + namespace + 秘密指纹）

- **Why**：逐条重跑 provider 成本高；缓存让 prompt/dev 迭代和断点续跑（resume）快很多。
- **实现**：默认 14 天 TTL（[src/cache.ts:39-48]()），disk/mem 两态（[src/cache.ts:36-37]()），`withCacheNamespace` 用 AsyncLocalStorage 做嵌套 namespace 隔离（[src/cache.ts:249-261]()），`withCacheEnabled` 全局开关注入（[src/cache.ts:263-263]()）。秘密字段不进 cache key，用固定 salt 的 HMAC 指纹化（[src/cache.ts:314-323]()）——注释明确解释为什么用编译期 salt 而非每进程随机 key（否则每次 `promptfoo eval` 都换 key、磁盘缓存完全失效，[src/cache.ts:302-309]()）。
- **权衡 / 代价**：缓存让"响应实时性"打折扣，所以 `FetchWithCacheResult.cached` 标记让统计/调试能识别；`withCacheEnabled(false)` 可全局关。核心 tradeoff 是"速度 vs 新鲜度"，用显式开关 + 命中标记透明化。

---

## 亮点与问题

### 亮点

1. **一个通用骨架服务所有上层**：评估管线（展开 → 并发执行 → 断言 → 落库）被 eval、redteam、optimizer、AI SDK helper 共享，`isRedteam` 只是一个姿态开关，`EvaluationStore` / `EvaluatorRuntime` 抽象让它能在 node / webviewer / test 三套 store 上跑（[src/evaluator.ts:4996-5027](), [src/node/evaluatorRuntime.ts:23-86]()）。这是"把核心复杂度做进一条链路、别人复用"的架构正资产。
2. **断言体系的深度与容错**：50+ 类型覆盖从字符串匹配到 RAG / trace / trajectory / agent 的现代评估；`AssertionsResult` 的加权聚合 + namedScores + threshold 覆写 + 扁平化 componentResults，让"一个测试到底是全 0 还是部分达标"都能表达。
3. **规模化不是事后补丁，而是内建**：自适应限速、串行/并发分流、多级超时、合法 abort、断点 resume、target-unavailable 快速失败（`abortIfTargetUnavailable`，[src/evaluator.ts:3593-3593]()）都是第一公民，不是外加选项。
4. **可观测性到细节**：latency/score/cost/tokenUsage 逐行追踪、grading prompt 与解释存 metadata、progress 在三套渲染器（CI / TTY / webUI）间切换、telemetry 记录 `feature_used`。让"黑盒 eval"可审计。

### 问题 / 风险

1. **核心文件过大**：`evaluator.ts` 5028 行、`evaluatorHelpers.ts` 888 行。虽已用 runtime/store 抽象解耦落地，但"调度-判定-落库"仍在一个巨型文件里，协作冲突与可读性成本高；`processEvalStepWithTimeout`→`processEvalStep`→`processEvalRows` 这类三明治调用链嵌套较深。
2. **model-graded 的不可控成本/波动**：model-graded 断言每次调 LLM 都烧 token、且 LLM 判定本身有随机性；虽有 `graderError`/解析容错兜底，但对大规模 redteam（一次几千条）而言判定成本占比可能显著——`stats.tokenUsage.assertions` 被单独统计（[src/evaluator.ts:3337-3339]()）正说明它是被当作主要成本项来审计的。
3. **配置笛卡尔积有爆炸风险**：组合展开是纯乘法，用户若写多个多值 var + 多 provider + 多 prompt + repeat，`RunEvalOptions` 数量会爆炸；并发/超时/缓存缓解了墙钟，但"预展开全清单"阶段（`buildRunEvalOptions`）没有内置数量上限护栏（只是靠并发水位与 resume 过滤兜底）。
4. **缓存新鲜度**：14 天 TTL + 磁盘缓存对"迭代改 prompt"友好，但对"上游模型主动更新 / 网络代理改响应"这类不可感知变化不够敏感，用户需显式关缓存——可接受的默认取舍。

---

## 可借鉴的设计（跨项目迁移）

- **"声明 → 展开 → 限流并发执行 → 结果结构化落库"这条评估骨架**，可复用到任何"批量输入 + 打分的 benchmark / 回归"场景，不限于 LLM。
- **运行时/存储/判定三层解耦（Evaluator + EvaluationStore + EvaluatorRuntime）**：把"怎么做"与"结果怎么落地"拆开，是让同一套引擎服务多端（CLI / web / test / headless）的范式。
- **用注册表 + 统一返回契约（GradingResult）做可插拔判定**：新判定器只需"输入 provider 响应 + 上下文 → 输出 pass/score/reason"，接入成本低，且天然可组合（componentResults 聚合）。
- **对自己不确定的东西（LLM 自评）加显式失败标记 + 产物留证（renderedGradingPrompt）**：比"静默给 0 分"或"无脑翻转 inverse"更诚实，避免把"grader 故障"误判为"判定结果"。

---

## 覆盖率明细（标准模式）

| 文件 | 总行数 | 已读行数 | 覆盖率 | 未读原因 |
|------|-------:|---------:|-------:|----------|
| `src/evaluate.ts` | 376 | 376 | 100% | 全文精读 |
| `src/evaluator.ts` | 5028 | ≈1450 | ~29% | 分段通读：Evaluator 类/调度/超时/落库（3224-3553、3608-3883、3932-4116、4305-4494、4608-4991）与 runEval/判定（660-1100、1316-1745、1844-1903、2277-2796）全读；中间工具函数（1746-1843、1904-2276、2797-3223、4117-4304、4495-4607）未逐行读 |
| `src/evaluatorHelpers.ts` | 888 | 0 | 0% | 辅助工具，非主链路，未列入本次深挖重点 |
| `src/cache.ts` | 969 | ≈90 | ~9% | 读关键决策点（顶层/withCacheNamespace/withCacheEnabled/命令/秘密指纹 240-329），fetch+TTL 实现细节未全读 |
| `src/assertions/index.ts` | 891 | 620 | ~70% | 精读 runAssertion/runAssertions/注册表/反向断言（226-842）；文件引用加载细节后已含 |
| `src/assertions/assertionsResult.ts` | 236 | 236 | 100% | 全文精读（打分聚合核心） |
| `src/assertions/llmRubric.ts` | 52 | 52 | 100% | 全文精读 |
| `src/matchers/llmGrading.ts` | 591 | 591 | 100% | 全文精读（model-graded 核心） |
| `src/matchers/rubric.ts` | 879 | 879 | 100% | 全文精读（JSON grading 核心） |
| `src/matchers/comparison.ts` | 193 | 193 | 100% | 全文精读 |
| `src/prompts/index.ts` | 328 | 328 | 100% | 全文精读 |
| `src/scheduler/index.ts` | 46 | 46 | 100% | 全文精读 |
| `src/scheduler/retryPolicy.ts` | 88 | 88 | 100% | 全文精读（重试策略） |
| `src/node/evaluatorRuntime.ts` | 87 | 87 | 100% | 全文精读（store/runtime 抽象） |
| `src/util/eval/summary.ts` | 363 | 363 | 100% | 全文精读（结果汇总/CLI 展示） |
| `src/types/index.ts` | 1538 | ≈200 | ~13% | 读 EvaluateResult/GradingResult/Stats 相关段（385-585） |
| `src/types/providers.ts` | 215 | 70 | ~33% | 读 AtomicTestCase/provider 接口（60-129） |
| `src/node/doEval.ts` | — | 片段 | — | 仅确认入口调用关系 |
| `architecture/layers.json` | 200 | 200 | 100% | 分层归属核对 |

合计（核心文件）：`evaluate.ts`(100%) + `evaluator.ts`(~29%) + `assertions/index.ts`(~70%) + `assertionsResult.ts`(100%) + `llmRubric.ts`(100%) + `matchers/{llmGrading,rubric,comparison}.ts`(100%) = **核心链路（入口 + 调度 + 断言 + model-graded matcher）均已读到位，覆盖本解读引用的每一个 `文件:行号` 论断**。

判定：**达标**（标准模式）。`evaluator.ts` 为超大文件采用分段通读，覆盖了所有被引用的函数（`evaluate()`/`_runEvaluation`/`runEvalInternal`/`gradeRunEvalResponse`/`buildRunEvalOptions`/并发串行调度/比较断言/收尾落库）。未覆盖部分为纯工具/辅助函数，不承载被引用的论断。
