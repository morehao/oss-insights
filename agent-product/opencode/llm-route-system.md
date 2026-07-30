# OpenCode LLM 四轴路由系统

> **关联文档:** [主解读](./README.md) · [系统架构深读](./architecture.md)

---

## 1. 设计动机

AI 编码工具需要支持多个 LLM Provider，而每个 Provider 的 API 在语义、传输、鉴权方面各有差异。传统的适配方式是"每个 Provider 一个适配器类"，这会导致大量重复代码——DeepSeek、TogetherAI、Cerebras 的 API 和 OpenAI Chat 几乎一致，但被各处复制。

OpenCode 的 `@opencode-ai/llm` 包采用**四轴分解**设计：将 Provider 调用拆解为四个正交维度，每一维可以独立替换和组合。

```
┌─────────────────────────────────────────────────────────────────┐
│                         Route.make()                            │
│                                                                 │
│   Protocol   +   Endpoint   +   Auth   +   Transport            │
│   (语义合约)     (URL 构造)     (鉴权)      (传输 + 帧)          │
│                                                                 │
│  OpenAI Chat    /chat/completions  Bearer       HTTP SSE        │
│  Anthropic      /v1/messages      x-api-key    HTTP SSE        │
│  Gemini         /v1/models/*      Bearer       HTTP SSE        │
│  Bedrock        /model/*/converse SigV4        Binary Stream   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 四轴详解

### 2.1 Protocol — 语义 API 合约

**文件:** `packages/llm/src/route/protocol.ts`

Protocol 定义了 Provider 的"语言"——Request body 的结构和 Response stream 的状态机：

```ts
interface Protocol<Body, Frame, Event, State> {
  id: ProtocolID
  body: ProtocolBody<Body>         // { schema, from }
  stream: ProtocolStream<Frame, Event, State>
}
```

| 组成部分 | 说明 | 示例 |
|---------|------|------|
| `body.schema` | Schema.Codec 校验 body 合法性 | `Schema.Struct({ model, messages, ... })` |
| `body.from` | 从通用 `LLMRequest` 构造 Provider 原生 body | 将 System/User/Assistant 转为 OpenAI 消息格式 |
| `stream.event` | Schema.Codec 解码一个 frame 为类型化事件 | SSE 数据行 → `{ type: "content_block_delta", ... }` |
| `stream.step` | 状态机: `(state, event) → [State, LLMEvent[]]` | 累积内容块、处理工具调用 |

**现有 Protocol：**

| Protocol | ID | 文件 | 行数 |
|---------|----|------|------|
| OpenAI Chat | `openai-chat` | `protocols/openai-chat.ts` | ~506 |
| OpenAI Responses | `openai-responses` | `protocols/openai-responses.ts` | 更多 |
| Anthropic Messages | `anthropic-messages` | `protocols/anthropic-messages.ts` | — |
| Gemini | `gemini` | `protocols/gemini.ts` | — |
| Bedrock Converse | `bedrock-converse` | `protocols/bedrock-converse.ts` | ~674 |

### 2.2 Endpoint — URL 构造

**文件:** `packages/llm/src/route/endpoint.ts`

Endpoint 提供声明式 URL 构造：

```ts
interface Endpoint<Body> {
  baseURL?: string
  path: EndpointPart<Body>         // string | ((input) => string)
  query?: Record<string, string>
}
```

**两种构造方式：**

| 方式 | 示例 | 使用场景 |
|------|------|---------|
| 静态路径 | `Endpoint.path("/chat/completions")` | OpenAI / Anthropic / Gemini |
| 动态函数 | `Endpoint.path(({ body }) => \`/model/${body.modelId}/converse-stream\`)` | Bedrock（URL 嵌入 modelId） |

### 2.3 Auth — 鉴权方案

**文件:** `packages/llm/src/route/auth.ts`

Auth 是一个 Effect 函数，接收请求参数返回 Headers：

```ts
interface Auth {
  apply: (input: AuthInput) => Effect<Headers, AuthError>
  andThen: (that: Auth) => Auth    // 组合: 两个都执行
  orElse: (that: Auth) => Auth     // 回退: 第一个失败则第二个
}
```

**内置的 Auth 构造器：**

| 构造器 | 效果 | 示例 |
|--------|------|------|
| `Auth.bearer(secret)` | `Authorization: Bearer <key>` | OpenAI |
| `Auth.header("x-api-key", secret)` | 自定义 Header | Anthropic |
| `Auth.config("ANTHROPIC_API_KEY")` | 读环境变量 | 通用 |
| `Auth.optional(secret, source)` | 可选凭据 | 可略过 |
| `Auth.custom(apply)` | 任意签名逻辑 | Bedrock SigV4 |
| `Auth.none` | 无认证 | 测试 |

**Provider 的鉴魂配置差异：**

| Provider | Auth 方案 |
|----------|----------|
| OpenAI | `Auth.bearer(apiKey)` 回退到 `OPENAI_API_KEY` |
| Anthropic | `Auth.header("x-api-key", apiKey)` 回退到 `ANTHROPIC_API_KEY` |
| Bedrock | `BedrockAuth.auth` — 完整 SigV4 请求签名 |
| GitHub Copilot | `AuthOptions.bearer(options)` — 无环境变量回退 |
| 兼容 Provider | `Auth.bearer(apiKey)` 回退到各 Provider 的环境变量 |

### 2.4 Transport — 传输与帧

**文件:** `packages/llm/src/route/transport/index.ts`

Transport 负责实际的网络通信：

```ts
interface Transport<Body, Prepared, Frame> {
  id: string
  prepare: (input) => Effect<Prepared, LLMError>
  frames: (prepared, request, runtime) => Stream<Frame, LLMError>
}
```

**两个 Transport 实现：**

| Transport | 文件 | 说明 |
|-----------|------|------|
| `HttpTransport.httpJson` | `transport/http.ts` | HTTP POST + 响应体流式读 |
| `WebSocketTransport.json` | `transport/websocket.ts` | WebSocket JSON 消息 |

**Framing — 字节流 → 帧流：**

```ts
interface Framing<Frame> {
  id: string
  frame: (bytes: Stream<Uint8Array>) => Stream<Stream<Frame>>
}
```

| Framing | 说明 | 使用 |
|---------|------|------|
| `Framing.sse` | Server-Sent Events | OpenAI / Anthropic / Gemini |
| `BedrockEventStream` | AWS event-stream 二进制帧 | Bedrock（含 CRC 校验） |

---

## 3. 组合方式

### 3.1 Route.make() 入口

**文件:** `packages/llm/src/route/client.ts`

```ts
// 标准 HTTP 路由
Route.make({
  id: "openai-chat",
  protocol: OpenAIChat.protocol,
  endpoint: Endpoint.path("/chat/completions"),
  auth: Auth.bearer(apiKey),
  framing: Framing.sse,
})

// 自定义 Transport 路由 (WebSocket)
Route.make({
  id: "openai-responses-ws",
  protocol: OpenAIResponses.protocol,
  endpoint: Endpoint.path("/realtime"),
  transport: WebSocketTransport.json(...),
})
```

### 3.2 5 行实现 OpenAI 兼容路由

**文件:** `packages/llm/src/protocols/openai-compatible-chat.ts:17-23`

```ts
export const route = Route.make({
  id: "openai-compatible-chat",
  protocol: OpenAIChat.protocol,        // ← 复用整个语义 API
  endpoint: Endpoint.path("/chat/completions"),
  framing: Framing.sse,
})
```

OpenCode 7+ 兼容 Provider（DeepSeek、TogetherAI、Cerebras、Groq、Fireworks、DeepInfra、Baseten）都共此路由。

### 3.3 Bedrock 动态路径示例

**文件:** `packages/llm/src/protocols/bedrock-converse.ts:658-670`

```ts
export const route = Route.make({
  id: "bedrock-converse",
  provider: "bedrock",
  protocol,
  endpoint: Endpoint.path<BedrockConverseBody>(
    ({ body }) => `/model/${encodeURIComponent(body.modelId)}/converse-stream`,
  ),
  auth: BedrockAuth.auth,     // ← SigV4 签名
  framing,                    // ← 自定义二进制帧
})
```

### 3.4 Route.with() 提供者适配

`Route.with(patch)` 创建重写了部分配置的新路由——每个 Provider 适配器就是用 `.with()` 做"局部配置"：

```ts
// OpenAI Provider ("openai.ts:37-56")
const configure = (input: Config = {}) => {
  const route = baseRoute.with({
    auth,
    endpoint: { baseURL, query },
  })
  return {
    model: (id) => route.with(defaults).model({ id }),
    responses: (id) => route.with(options).model({ id }),
    configure,
  }
}
```

---

## 4. Provider 注册体系

Provider 不注册到中央注册表，而是通过 **barrel 导出 + 按需导入**的模式：

### 4.1 Provider 文件结构

**目录:** `packages/llm/src/providers/`

```
providers/
├── openai.ts              → OpenAI.chat("gpt-4o")
├── anthropic.ts           → Anthropic.chat("claude-3.5-sonnet")
├── amazon-bedrock.ts      → Bedrock.chat("us.anthropic.claude-3-5-sonnet")
├── azure.ts               → Azure.chat("gpt-4o")
├── google.ts              → Google.chat("gemini-2.0-flash")
├── cloudflare.ts          → CloudflareAIGateway / CloudflareWorkersAI
├── github-copilot.ts      → GitHub Copilot
├── xai.ts                 → xAI
├── openrouter.ts          → OpenRouter
├── openai-compatible.ts   → 通用兼容 Provider 工厂
└── openai-compatible-profile.ts → 9 个 Provider Profile
```

### 4.2 复用链

从 Protocol 到具体 Provider 的完整链路：

```
OpenAIChat.protocol (语义 API)

    └── openai-compatible-chat.ts (5行路由定义)

            └── openai-compatible.ts (Provider 工厂)

                    └── openai-compatible-profile.ts (Profile 定义)

                            ├── deepseek: { baseURL: "https://api.deepseek.com/v1" }
                            ├── cerebras: { baseURL: "https://api.cerebras.ai/v1" }
                            ├── groq: { baseURL: "https://api.groq.com/openai/v1" }
                            ├── togetherai: { baseURL: "https://api.together.xyz/v1" }
                            ├── deepinfra: { baseURL: "https://api.deepinfra.com/v1/openai" }
                            ├── fireworks: { baseURL: "https://api.fireworks.ai/inference/v1" }
                            ├── baseten: { baseURL: "https://bridge.baseten.co/v1" }
                            ├── openrouter: { baseURL: "https://openrouter.ai/api/v1" }
                            └── xai: { baseURL: "https://api.x.ai/v1" }
```

**Profile 的定义** (`packages/llm/src/providers/openai-compatible-profile.ts`):

```ts
const profiles = {
  deepseek:    { provider: "deepseek",    baseURL: "https://api.deepseek.com/v1" },
  cerebras:    { provider: "cerebras",    baseURL: "https://api.cerebras.ai/v1" },
  groq:        { provider: "groq",        baseURL: "https://api.groq.com/openai/v1" },
  // ...
} as const
```

### 4.3 新增一个 Provider 的完整流程

以新增 `xyz-ai`（OpenAI 兼容）为例：

**步骤 1:** 在 profile 中添加一行
```ts
xyz: { provider: "xyz-ai", baseURL: "https://api.xyz.ai/v1" }
```

**步骤 2:** 在 barrel 中导出
```ts
export const xyz = () => OpenAICompatible.define(profiles.xyz)
```

**完成。** 4 行代码新增一个 Provider（如果已有环境变量规范，再加 1 行）。

---

## 5. 执行流水线

```
LLMClient.stream(request)  或  LLMClient.generate(request)
    │
    ▼
compile(request)
    │
    ├── 1. resolveRequestOptions
    │      合并: route.defaults > model.defaults > request.options
    │
    ├── 2. route.body.from(request)
    │      将通用 LLMRequest 转为 Provider 原生格式
    │
    ├── 3. validateWith(body.schema)
    │      Schema.Codec 校验，确保 body 合法
    │
    ├── 4. route.prepareTransport(body, request)
    │      构建 HTTP Request (URL + Headers + Body)
    │      └─ Auth.apply() → 注入鉴权头
    │
    ▼
streamPrepared
    │
    ├── 5. transport.frames(prepared, request, runtime)
    │      HTTP POST → 字节流
    │      └─ Framing.frame → Stream<Frame>
    │
    ├── 6. decodeEvent(frame)
    │      用 stream.event Schema 解码一帧为类型化事件
    │
    └── 7. stream.step(state, event)
          状态机: [State, LLMEvent[]]
          └─ 输出: TextDelta / ToolCall / StepFinish / Error
```

---

## 6. 设计评价

### 亮点

| 方面 | 评价 |
|------|------|
| **接口隔离** | 四轴正交分解使每个维度可独立演化和替换 |
| **组合性** | Route.with() 让 Provider 定制保持在"局部配置"级别 |
| **复用度** | 7+ Provider 共享 1 个 Protocol，新增 Provider 成本极低 |
| **类型安全** | 全套 Schema Codec 校验 + 泛型参数保证编译期类型一致 |
| **架构可扩展** | 添加新的 API 协议（如新增 OpenAI Responses）只需新增 Protocol 定义 |

### 与其他项目的对比

| 项目 | LLM 适配方式 | 新增 Provider 成本 |
|------|-------------|-------------------|
| **OpenCode** | 四轴可组合路由 | 5 行（OpenAI 兼容）/ ~80 行（全新协议） |
| **LangChain** | 抽象基类 + 继承 | 500-1000 行（每个 Provider 独立实现） |
| **Vercel AI SDK** | Provider 独立包 | 每个 Provider 写完整适配 |

OpenCode 的设计在 Provider 适配的**开发成本**和**维护一致性**上显著优于 LangChain 的类继承方式，核心原因在于四轴分解使得 Provider 之间的差异被显式表达为"在哪个轴上有何不同"，而非整体重写。

### 潜在改进

- Profile 体系目前是静态配置，未来可支持动态 Provider 发现
- WebSocket Transport 的复用场景有限（仅 OpenAI Responses 使用）
- Framing 目前仅 SSE 和 Bedrock 两种，如需支持 gRPC 流需要扩展

---

> **关联文档:** [主解读](./README.md) · [系统架构深读](./architecture.md)
