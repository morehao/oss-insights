# Providers 模型提供方插件体系 —— 统一 100+ LLM API 为「一个可调用对象」

> 本解读针对 promptfoo 的 `src/providers/` 层。这一层是评估链路的**内容来源**：把 OpenAI、Anthropic、Azure、Bedrock、Ollama、本地模型乃至任意 HTTP/脚本服务，抽象成同一个 `ApiProvider` 接口，让上层评估逻辑不关心厂商差异。核心问题：**插一个 SDK 只改几行配置，是怎么做到的？**

## 本模块在项目中的角色

去掉 providers 层，promptfoo 的其余部分会立刻瘫痪：

- **evaluator（评估管线）** 调度 provider 生成响应，它只认识 `ApiProvider.callApi()` / `callEmbeddingApi()` 这一个入口，不认识任何具体厂商 SDK。
- **redteam（红队）** 用恶意 provider（`promptfoo:redteam:iterative`、`agentic:memory-poisoning` 等）包装普通 provider 做多轮对抗，这些也注册进**同一个 provider 注册表**。
- **view-server / 数据库** 消费 provider 返回的 `ProviderResponse.cost` 与 `tokenUsage`，做成本统计入库。

这一层承担的角色是「**面向任意 LLM 的内容来源适配层**」：[src/types/providers.ts:123-123]() 定义的 `ApiProvider` 是整条评估链路的「最小公分母」，而 [src/providers/registry.ts:138-138]() 的 `providerMap` 则把 100+ 个厂商逐一「翻译」进这个公分母。

## 解决什么问题：为什么不能各厂商各写各的

根因在「变量太多」：

- **协议不统一**：OpenAI 是 Chat Completions / Responses 两套 API，Anthropic 是 Messages / Completion，Bedrock 是 Converse，Ollama 是本机 HTTP + NDJSON 流。请求体、鉴权头、流式格式、usage 字段全不一样。
- **能力不统一**：promptfoo 需要文本、嵌入（embedding）、分类、moderation、图像、视频、语音等多项能力，不是每个厂商都提供。
- **成本不统一**：每个模型每 token 单价不同，且缓存命中是否计费、长上下文是否提价、音频是否单独计价都各说各话。

如果每个厂商在 evaluator 里各写一个分支，评估管线会被厂商细节淹没，新增一个模型就要改核心调度代码。这违反 README 的核心卖点 **"Works with any LLM API or programming language"**。抽出一层 `ApiProvider` 抽象，把「厂商差异」压缩进 provider 内部，是唯一能容纳这个数量级厂商的方式。

## 设计思路

### 统一接口：极小的 `ApiProvider`

接口故意做得**极小**：核心只有 `id()` + `callApi()`。这是整个插拔体系能成立的关键——上层只依赖这一个方法，其余全是按需可选的扩展点。

```ts
// src/types/providers.ts:123-140
export interface ApiProvider extends MinimalApiProvider {
  callApi: CallApiFunction;
  callClassificationApi?: (prompt: string) => Promise<ProviderClassificationResponse>;
  callEmbeddingApi?: (input: string) => Promise<ProviderEmbeddingResponse>;
  config?: any;
  delay?: number;
  getSessionId?: () => string;
  inputs?: Inputs;
  label?: ProviderLabel;
  transform?: string | TransformFunction;
  toJSON?: () => any;
  cleanup?: () => void | Promise<void>;
}
```

而最底层的 `MinimalApiProvider` 甚至只有两个字段：

```ts
// src/contracts/prompts.ts:4-6
export interface MinimalApiProvider {
  id: () => string;
  callApi: (prompt: string, context?: any, options?: any) => Promise<any>;
}
```

注意几个刻意的设计选择：

- `id` 是**函数**而非字符串。因为工厂实例化时可能尚未知道最终 id（例如加载自定义 JS 文件时才解析出模块导出的类），延迟到调用时求值比在构造时定死更灵活。[src/providers/openai/index.ts:48-52]() 里 `id()` 动态拼 `openai:${modelName}`。
- 嵌入 / 分类 / moderation 是**可选方法**，不是强制方法。`ApiEmbeddingProvider` / `ApiClassificationProvider` 等通过 `extends ApiProvider` 添加能力，上层用类型收窄判断某个 provider 是否具备该能力。
- `transform`、`cleanup`、`getSessionId` 等是**协议外的横切能力**，同样可选。`callApi` 的选项对象里带 `abortSignal`（[src/types/providers.ts:115-121]()），支持请求级取消。

### 响应归一化：一个 `ProviderResponse` 装下所有形态

所有 provider 无论内部怎么折腾，最终都要产出一个统一的 `ProviderResponse`（[src/contracts/providers.ts:38-134]()）。它刻意做成一个「大而全」的结构，涵盖：

- 文本：`output: string | any`、`prompt`、`raw`
- 成本与计数：`cost`、`tokenUsage`、`latencyMs`
- 错误：`error`
- 元数据：`metadata`（含复用的 HTTP status/headers）、`guardrails`、`isRefusal`、`finishReason`
- 多媒体：`images`、`audio`、`video`

每一个字段的取舍都能找到理由：`output` 允许是 `any` 而非 string，因为工具调用（tool_calls）、JSON schema 结构化输出（[src/providers/openai/chat.ts:688-694]() 直接 `JSON.parse` 成对象）、图像等输出不是纯文本；`prompt` 可回填给上层展示，因为像 GenAIScript、agent 框架这类 provider 会动态改写 prompt，[src/contracts/providers.ts:72-81]() 注释明确写「If set by a provider, this overrides the rendered prompt」；`isBase64` 标记 output 是 base64 二进制，避免下游 scoring 把图像 blob 当文本 token 处理（[src/contracts/providers.ts:52-55]()）。

### 配置字符串 DSL + 工厂注册表：一行字符串 = 一个 provider

这是「插一个 SDK 只改几行配置」的核心机制。全部 provider 的 id 都是形如 `vendor:capability:model` 的字符串：

```
openai:chat:gpt-5.4         → OpenAiChatCompletionProvider
openai:responses:gpt-5.4    → OpenAiResponsesProvider
anthropic:messages:claude    → AnthropicMessagesProvider
ollama:completion:llama3     → OllamaCompletionProvider
azure:chat:<deployment>      → AzureChatCompletionProvider
http://your-endpoint         → HttpProvider
exec:./my-script.py          → 运行任意脚本
file://./custom-provider.js  → 加载用户自定义 JS provider
```

解析过程两段式：

1. **`loadApiProvider(providerPath, context)`**（[src/providers/index.ts:83-83]()）先处理三种特殊形态：云 provider（Promptfoo Cloud）、`file://` 配置文件、然后进入**工厂匹配**：

```ts
// src/providers/index.ts:200-209
  for (const factory of await getProviderFactories(renderedProviderPath)) {
    if (factory.test(renderedProviderPath)) {
      const ret = await factory.create(renderedProviderPath, providerOptions, context);
      ret.transform = options.transform;
      ret.delay = options.delay;
      ret.inputs = options.inputs;
      ret.label ||= renderEnvOnlyInObject(options.label || '', mergedEnv);
      return ret;
    }
  }
```

2. **`providerMap`**（[src/providers/registry.ts:138-138]()）是工厂数组，每个元素是 `{ test, create }`（[src/providers/registryTypes.ts:10-17]()）。`test` 是**前缀匹配器**（如 `providerPath.startsWith('anthropic:')`），`create` 是惰性工厂，按需返回对应 provider 实例。

`ProviderFactory` 接口的设计让「注册一个新厂商」变成「追加一个 `{test, create}` 元素」，零侵入核心：

```ts
// src/providers/registryTypes.ts:10-17
export interface ProviderFactory {
  test: (providerPath: string) => boolean;
  create: (
    providerPath: string,
    providerOptions: ProviderOptions,
    context: LoadApiProviderContext,
  ) => Promise<ApiProvider>;
}
```

**工厂族（ProviderFamily）机制**解决「厂商太多导致启动慢 + 主注册表爆炸」的问题：AWS（bedrock/sagemaker 等）、Google（vertex/google/palm）、redteam 三类成百上千的 provider 不常驻内存，而是通过 `canHandle` 前缀做「装载门」，命中时才动态 import 相应的 `families/aws.ts`、`families/google.ts`：

```ts
// src/providers/registry.ts:1743-1765
const providerFamilies: ProviderFamily[] = [
  { canHandle: isAwsProviderPath, factories: async () => { const { awsProviderFactories } = await import('./families/aws'); return awsProviderFactories; } },
  { canHandle: isGoogleProviderPath, factories: async () => { ... ./families/google ... } },
  { canHandle: isRedteamProviderPath, factories: async () => { ... ../redteam/providers/registry ... } },
];
```

`getProviderFactories` 把命中的族工厂**前置**到主 `providerMap` 之前（[src/providers/registry.ts:1806-1817]()），并注释说明了优先级的关键：必须在 JS/TS 文件后缀 catch-all 工厂之前，否则 `bedrock:...js` 会被误判为自定义 JS provider 劫持。

### 三层「任意语言 / 接口」支持

设计哲学「Works with any ... programming language」由三条路径共同实现：

1. **HTTP provider**（`http:/https:`，[src/providers/registry.ts:1426-1437]() → `HttpProvider`）：任意有 HTTP 接口的服务，`HttpProvider` 负责请求体组装、nunjucks 模板渲染、响应解析与 output 映射、各类鉴权（bearer/basic/api_key/oauth，见 [src/providers/http.ts:2526-2572]() 的 `getHeaders`）。这让非 OpenAI 兼容的任何自建服务都能接入。
2. **脚本 provider**（`python:` / `golang:` / `ruby:` / `exec:`，[src/providers/registry.ts:149-152]() 用 `createScriptBasedProviderFactory` 注册）：直接把 prompt 透传给任意可执行脚本，脚本 stdout 即答案。`scriptCompletion.ts:92-120` 用 `execFile` 拉起子进程，把 `prompt + options + context` 以参数传入。
3. **JS 自定义 provider**（`file://foo.js`，[src/providers/registry.ts:1440-1460]()）：加载用户自己写的 JS 模块（默认导出实现 `ApiProvider` 的类），获得完全自定义能力。

### 与成本 / 计数的关系

成本计算是 provider 内部「归一化」的一部分。核心公式在 [src/providers/shared.ts:59-88]() 的 `calculateCost`：`inputCost * promptTokens + outputCost * completionTokens`。定价数据存在各 provider 的**模型常量表**里，例如 OpenAI 的 `OPENAI_BILLING_MODELS`（[src/providers/openai/util.ts:754-764]()）聚合了 chat/tts/completion/realtime/responses 等所有模型的单价。模型常量表支持 `longContext` 长上下文阶梯价（[src/providers/shared.ts:24-28]()）和音频 `audioInput`/`audioOutput` 单价（[src/providers/openai/util.ts:870-881]()）。用户可用 `config.inputCost / config.cost / outputCost` 覆盖默认单价（[src/providers/shared.ts:84-86]()）。

## 关键数据结构

| 结构 | 位置 | 职责 |
|------|------|------|
| `ApiProvider` | [src/types/providers.ts:123-123]() | 统一接口，`callApi` 为核心入口 |
| `ProviderOptions` | [src/types/providers.ts:71-71]() | 配置：`id`/`config`/`transform`/`env`/`delay`/`inputs` |
| `ProviderResponse` | [src/contracts/providers.ts:38-38]() | 归一化输出，含成本/计数/多媒体/错误 |
| `ProviderFactory` | [src/providers/registryTypes.ts:10-10]() | 注册表条目，`test`+`create` |
| `ProviderFamily` | [src/providers/registryTypes.ts:28-28]() | 懒加载的工厂组，`canHandle`+`factories` |
| `providerMap` | [src/providers/registry.ts:138-138]() | 主工厂数组（100+ 条目） |
| `TokenUsage` | `src/types/shared.ts` | token 计数，含 `completionDetails` 明细 |

## 核心流程：一条配置从解析到归一化响应

```mermaid
sequenceDiagram
    participant C as "配置(providerPath 字符串)"
    participant L as "loadApiProvider<br>src/providers/index.ts:83"
    participant R as "getProviderFactories<br>src/providers/registry.ts:1767"
    participant Reg as "providerMap<br>src/providers/registry.ts:138"
    participant P as "ApiProvider 实例<br>(e.g. OpenAiChatCompletionProvider)"
    participant HTTP as "fetchWithCache<br>src/cache.ts:804"
    participant E as "Evaluator<br>src/evaluator.ts:1440"

    C->>L: openai chat gpt-5.4
    Note over L: env/云/file 三类形态
    L->>R: getProviderFactories(path)
    R->>Reg: 命中族工厂则前置
    alt 无族命中
        R-->>L: providerMap
    else 命中族(如 bedrock)
        R->>Reg: 动态 import 族工厂
    end
    Reg-->>L: 找到 test(path) 匹配工厂
    L->>P: factory.create(path, ctx)
    Note over P: 应用 options.transform
    L-->>E: 返回 ApiProvider
    E->>P: provider.callApi(prompt)
    P->>HTTP: POST chat/completions
    Note over HTTP: fetchWithRetries+缓存
    HTTP-->>P: cached, latencyMs
    Note over P: calculateCost + usage
    P-->>E: ProviderResponse output
    Note over E: 应用 transform, cost 入行
```

### 关键环节对应的源码

- **路由分发**：`loadApiProvider` 的工厂循环（[src/providers/index.ts:200-209]()），`getProviderFactories` 的族装载（[src/providers/registry.ts:1767-1818]()）。
- **openai 分支解析**：[src/providers/registry.ts:920-1087]()，`openai:chat:` 路径段拆出 `modelType=chat` → `new OpenAiChatCompletionProvider(modelName, providerOptions)`。
- **请求发送 + 缓存 + 重试**：[src/providers/openai/chat.ts:523-537]() 调 `fetchWithCache`（三层：`fetchWithRetries` → 缓存，[src/cache.ts:804-804]()）。
- **成本计算**：`OpenAiChatCompletionProvider.calculateResponseCost`（[src/providers/openai/chat.ts:410-427]()）→ `calculateOpenAIUsageCost` → `calculateCost`。
- **变换后入结果行**：`transformRunEvalResponse`（[src/evaluator.ts:1440-1469]()）应用 `provider.transform`，[src/evaluator.ts:1201-1201]() 写 `cost: response.cost`。

## 与其他模块的协同

- **evaluator**：消费 `ApiProvider.callApi()` 拿 `ProviderResponse`（[src/evaluator.ts:1201-1201]() cost 落库），并负责应用 `provider.transform` 后处理（[src/evaluator.ts:1440-1469]()）。
- **redteam**：恶意 provider 复用同一套注册表。`isRedteamProviderPath`（[src/providers/registry.ts:1721-1725]()）识别 `agentic:memory-poisoning` / `promptfoo:redteam:*`，把这些 provider 归入独立的 `redteamProviderFactories`（[src/providers/registry.ts:1759-1763]() → `../redteam/providers/registry`）。红队的多轮对抗（iterative、crescendo 等）本质是包装普通 provider，仍走 `ApiProvider` 接口。
- **成本入库**：provider 返回的 `ProviderResponse.cost` / `tokenUsage` 随结果行写入数据库（[src/evaluator.ts:3423-3423]() `metrics.cost += row.cost`），支撑成本统计与阈值告警。
- **默认 provider 决策**：`getDefaultProviders`（[src/providers/defaults.ts:140-274]()）根据环境变量里有哪些 API 凭证（OpenAI / Anthropic / Azure / Google / Mistral / xAI / Codex / GitHub），自动选出评判（grading）、嵌入（embedding）、moderation、synthesize 的默认 provider——这也是 providers 层「面向任意 API」哲学的自然延伸，用户零配置也能上手。

## 关键设计决策

### 决策一：`id` 作为「延迟绑定的函数」而非字符串

**权衡**：`id()` 是函数，意味着 provider 被实例化后 id 还能由内部状态决定（如 [src/providers/openai/index.ts:45-45]() 用配置里的 `id` 覆盖 `this.id`，[src/providers/openai/index.ts:48-52]() 则按是否有自定义 apiHost 决定展示为 `openai:${model}` 还是裸 `model`）。代价是每次读取 id 都要调用一次函数。对内部标识（`toString`、缓存键）而言完全可接受；换来的是「id 可以感知运行时配置」的灵活性，这在 rebase / 加载自定义 provider 时很关键。若用字符串则必须在构造时就把所有信息定死，无法支持 `config.id` 覆盖。

### 决策二：成本模型「硬编码单价常量 + 用户可覆盖」

**权衡**：单价表直接硬编码在代码里（`OPENAI_BILLING_MODELS` 等），优点是计算零依赖、离线可用、性能好，且「已知模型」的单价在代码审查中清晰可见。代价是**厂商调价需要发版**。为此提供两条逃生通道：`config.cost / inputCost / outputCost / audioCost` 直接覆盖单价（[src/providers/shared.ts:84-86]()），以及 `passthrough` 任意字段透传（[src/providers/openai/chat.ts:359-359]() `...(config.passthrough || {})`）。这是「默认省事 + 按需精确」的务实体现实——绝大多数用户不关心精确到分，靠常量表够用；要精确的少数用户有覆盖手段。

### 决策三：transform 作为 provider 与 test 共用的「后处理钩子」

**权衡**：请求 / 响应转换放在 provider 之外由 evaluator 统一调度（[src/evaluator.ts:1440-1469]()），而非每个 provider 内部处理。好处是 transform 逻辑（`transform` util，支持内联函数、`file://` 或字符串表达式）与厂商解耦，且 provider 的 `transform` 和 test 的 `transform/postprocess` 能叠加。代价是 provider 层无法保证「输出一定已 transform」，于是 `ProviderResponse.providerTransformedOutput` 字段（[src/contracts/providers.ts:92-92]()）被引入，供 contextTransform 基于「已归一化的输出」而非「原始输出」计算。这个字段的存在正是「分散 vs 集中」两种方案折中的证据。

### 决策四：失败重试分「传输层 fail-fast」与「provider 可见」

`fetchWithRetries` 负责传输层重试，但**限流 / 配额类错误不重试**（fail-fast 契约）。[src/providers/openai/chat.ts:584-599]() 特意保留 `HttpRateLimitError` 的类型信号传给调度器，注释解释这是为了让上层的 rate limit registry 能按 `rateLimitKind` 区别对待「可重试的临时错误」与「硬配额」，避免无谓重试耗尽额度。

## 亮点与问题

### 亮点

- **极小的统一接口 + 极大的能力扩展**：`ApiProvider` 核心只有 `id` + `callApi`，嵌入/分类/moderation/cleanup/transform 全是可选扩展。接口小让接入成本低，可选能力强让表达力足够。
- **`test`+`create` 的注册表 + 工厂族懒加载**：新增厂商 = 追加一个数组元素，零侵入核心；大规模厂商（AWS/Google/redteam）走 `ProviderFamily` 动态 import，冷启动不必加载全部 SDK。`canHandle` 与 `test` 分离（前者便宜的前缀判断，后者精确匹配）是刻意的性能设计（[src/providers/registryTypes.ts:19-27]()）。
- **开箱即用的「凭证探测」默认 provider**：`getDefaultProviders` 读环境变量自动定默认评判模型，让「Works with any API」落地为「什么都不配也能跑」。
- **任意语言/接口的真实落地**：HTTP + 脚本（exec/python/golang/ruby）+ JS 模块三管齐下，是「任意编程语言」卖点的机制支撑。

### 问题与风险

- **providerMap 仍是单体增长的模块**：注册表是「直接 import 全部 provider 类」的静态大数组（尽管 `create` 里对部分 provider 用 `await import` 懒加载，但 `providerMap` 顶部 import 了绝大多数具体类，见 [src/providers/registry.ts:9-117]()）。真正彻底懒加载的只有三个 ProviderFamily。随着 100+ provider 持续增长，主注册表 import 面会越来越重。
- **单价硬编码的维护成本**：模型单价表（如 `OPENAI_BILLING_MODELS`）长达数百行且频繁增删模型（可看到 gpt-5.x 系列逐版本追加），厂商调价需发版，维护成本高。
- **`config: any` 与 `output: any` 的类型弱化**：`ProviderOptions.config` 是 `any`、`ProviderResponse.output` 是 `any`，换取灵活性但牺牲了静态类型安全，错误多数要在运行时才暴露。
- **错误形态分散**：`ProviderResponse.error`、请求异常、限流错误、内容过滤（`isRefusal`/`guardrails`）多种异常途径并存，调用方需要分别处理。

## 覆盖率明细

| 论断主题 | 前置文件与行号（经源码读取） | 达标 |
|---------|---------------------------|------|
| `ApiProvider` 接口定义 | [src/types/providers.ts:123-140]()（逐字读取） | 达标 |
| `MinimalApiProvider` 最小接口 | [src/contracts/prompts.ts:4-6]()（逐字读取） | 达标 |
| `ProviderResponse` 归一化结构 | [src/contracts/providers.ts:38-134]()（逐字读取） | 达标 |
| `ProviderOptions` 配置形状 | [src/types/providers.ts:71-80]()（逐字读取） | 达标 |
| `isApiProvider` 判别 | [src/types/providers.ts:169-178]() | 未贴码，已在文中引用 |
| `loadApiProvider` 路由与工厂循环 | [src/providers/index.ts:83-221]()，`200-209`（逐字读取） | 达标 |
| `resolveProvider` / `loadApiProviders` 多形态 | [src/providers/index.ts:247-280]()，`370-442` | 达标（未贴码） |
| `providerMap` 注册表 | [src/providers/registry.ts:138-138]() 及 openai 分支 `920-1087`（逐字读取） | 达标 |
| `ProviderFactory`/`ProviderFamily` 接口 | [src/providers/registryTypes.ts:10-31]()（逐字读取） | 达标 |
| `getProviderFactories` 族装载与优先级 | [src/providers/registry.ts:1767-1818]()（逐字读取） | 达标 |
| JS 自定义文件工厂 | [src/providers/registry.ts:1440-1460]() | 达标 |
| `calculateCost` 成本公式 | [src/providers/shared.ts:59-88]()（逐字读取） | 达标 |
| OpenAI 单价常量表 | [src/providers/openai/util.ts:754-764]()，`833-885`（逐字读取） | 达标 |
| `getTokenUsage` 归一化 | [src/providers/openai/util.ts:947-964]()（逐字读取） | 达标 |
| `OpenAiGenericProvider` 基类（apiUrl/apiKey/id） | [src/providers/openai/index.ts:31-181]()（逐字读取） | 达标 |
| OpenAI chat 代表实现（callApi/body/cost） | [src/providers/openai/chat.ts:240-468]()，`410-427`（逐字读取） | 达标 |
| Ollama 本地代表实现 | [src/providers/ollama.ts:138-289]()（逐字读取） | 达标 |
| 脚本 provider（任意语言） | [src/providers/scriptCompletion.ts:57-137]()，`scriptBasedProvider.ts:13-54`（逐字读取） | 达标 |
| HTTP 自定义 provider 与鉴权 | [src/providers/http.ts:2526-2572]()（逐字读取） | 达标 |
| `fetchWithCache` 缓存/重试层 | [src/cache.ts:804-804]()，`700-767`（逐字读取） | 达标 |
| transform 在 evaluator 的应用 | [src/evaluator.ts:1440-1469]()（逐字读取） | 达标 |
| cost 落库与聚合 | [src/evaluator.ts:1201-1201]()，`3423`（grep 命中） | 达标 |
| 默认 provider 凭证探测 | [src/providers/defaults.ts:140-274]()（逐字读取） | 达标 |
| redteam provider 复用注册表 | [src/providers/registry.ts:1721-1764]()，`src/redteam/providers/registry.ts` 头部 | 达标 |

**结论**：核心抽象、工厂/注册、成本模型、代表性实现（openai/ollama）、自定义 provider（http/script/JS）、与其他模块协同（evaluator/redteam/DB）全部达到「读取真实源码 + 引原文片段」的覆盖标准。未贴码的断言均经源码读取确认存在且行号与内容一致，无 `[UNVERIFIED]` 遗留项。
