# OpenCode 自定义供应商 + 模型配置指南

> **关联文档:** [主解读](./README.md) · [LLM 四轴路由系统](./llm-route-system.md)
>
> 定位：面向需要接入**自有或第三方 OpenAI 兼容 API**（各类中转站、自建网关、企业私有端点）的用户，说明如何在 `opencode.json` 里配置一个自定义 Provider 与模型，并正确启用工具调用能力。

---

## 1. 入门：最小可用配置

OpenCode 通过 `opencode.json`（或 `.opencode/opencode.json`）的 `provider` 字段挂载自定义供应商。一个能跑通的最小配置段如下：

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "mygw": {
      "name": "My Gateway",
      "options": {
        "baseURL": "https://api.example.com/v3"
      },
      "models": {
        "my-model": { "name": "my-model" }
      }
    }
  }
}
```

三个必配项：

| 项 | 说明 |
|----|------|
| **provider 键名**（`mygw`） | 该供应商的唯一 ID，之后用 `mygw/<模型>` 引用 |
| **`options.baseURL`** | 服务端 OpenAI 兼容端点 |
| **`models`** | 定义要在该供应商下能选的模型 |

> 配置在 opencode 启动时一次性加载，**不热加载**。修改后需**退出并重启 opencode** 才生效。

---

## 2. models：每个模型有哪些字段

`models` 是一个对象，**键就是模型 ID**，值为该模型的元数据。下表为全部字段（类型 / 作用 / 默认值 / 是否必填）：

| 字段 | 类型 | 作用 | 默认值 | 必填 |
|------|------|------|--------|------|
| `id` | string | **实际下发到请求的 `model` 参数** | 等于 models 的键 | 视服务端 |
| `name` | string | 下拉 / 列表里的显示名 | 等于 models 的键 | 否 |
| `family` | string | 分组展示用（如 claude、deepseek） | 无 | 否 |
| `tool_call` | boolean | "支持工具调用"的能力标记 | `true` | 否 |
| `reasoning` | boolean | 是否为推理类模型（**仅标注**，不会自动开启请求侧推理，见「第 6 节」） | `false` | 否 |
| `interleaved` | string/obj | 推理流字段名（`reasoning`/`reasoning_content`/`reasoning_text`） | openai-compatible 下 deepseek 自动 `reasoning_content` | 否 |
| `temperature` | boolean | 是否支持 temperature 参数 | `false` | 否 |
| `attachment` | boolean | 是否支持附件/文件 | `false` | 否 |
| `release_date` | string | 发布日期，仅展示 | 无 | 否 |
| `status` | string | `active`/`beta`/`deprecated`/`alpha` | `active` | 否 |
| `modalities` | object | 输入/输出模态（`text`/`image`/`audio`/`video`/`pdf`） | input `[]`、output `[]` | 否 |
| `options` | object | 透传给请求体的额外配置 | 无 | 否 |
| `headers` | object | 追加的自定义请求头 | 无 | 否 |
| `cost` | object | 每百万 token 成本（input/output/cache），仅统计展示 | 无 | 否 |
| `limit` | object | 上下文窗口与输出上限（`context`/`input`/`output`） | `context 0`、`output 0` | 建议 |
| `variants` | object | 变体（如不同的 reasoning effort） | 无 | 否 |
| `provider` | object | 指定实现用的 npm 包 / api | 继承供应商 | 否 |

一个带较多字段的示例：

```jsonc
"models": {
  "deepseek-v4-flash": {
    "id": "deepseek/deepseek-v4-flash",
    "name": "deepseek-v4-flash",
    "tool_call": true,
    "reasoning": true,
    "limit": { "context": 128000, "output": 8192 },
    "modalities": { "input": ["text"], "output": ["text"] }
  }
}
```

---

## 3. provider 级字段

`provider.<name>` 本身也有一组配置项：

| 字段 | 类型 | 作用 |
|------|------|------|
| `name` | string | 展示名 |
| `npm` | string | 用哪个 AI SDK 包实现（如 `@ai-sdk/openai-compatible`）。**不写则默认按 openai-compatible 处理** |
| `env` | string[] | 从环境变量读取 API key（如 `["MYGW_API_KEY"]`） |
| `options` | object | 供应商级请求配置 |
| `models` | object | 该供应商下的模型集合（见上节） |

`options` 支持项：

```jsonc
"options": {
  "baseURL": "https://api.example.com/v3",
  "apiKey": "sk-...",                  // 或改用 env 注入
  "timeout": 120000,                     // 整请求超时，ms；false 表示关闭
  "headerTimeout": 30000,                // 等待响应头超时，ms
  "chunkTimeout": 30000                  // 流式 chunk 间隔超时，ms
}
```

> 认证二选一：把 key 直接写 `options.apiKey`，或用 `env` 引用环境变量（避免明文入库）。

---

## 4. 关键默认值：不配置时表现如何

不配置某个字段时，opencode 会从模型空对象（`@opencode-ai/schema` 的 `Model.Info.empty`）和运行时归集逻辑（`packages/opencode/src/provider/provider.ts`）取默认值。对你这样**无 `npm` 的自定义供应商**，实际生效的默认能力是：

```ts
// provider/provider.ts:1464
capabilities: {
  temperature: model.temperature ?? ... ?? false,
  reasoning:   model.reasoning   ?? ... ?? false,
  attachment:  model.attachment  ?? ... ?? false,
  toolcall:    model.tool_call   ?? ... ?? true,   // ← 工具默认开启
  input:  { text: true, audio/image/video/pdf: false },
  output: { text: true, audio/image/video/pdf: false },
}
limit: {
  context: model.limit?.context ?? ... ?? 0,   // ← 默认 0
  output:  model.limit?.output  ?? ... ?? 0,   // ← 默认 0
}
```

容易踩的默认值：

| 字段 | 默认 | 影响 |
|------|------|------|
| `tool_call` | `true` | 工具能力默认开启，**不需要专门写 true** |
| `limit.context` / `limit.output` | `0` | **建议显式配置**，否则上下文/输出可能被当 0 处理 |
| `modalities` | input `[]` / output `[]` | 多模态不展示；其中 text 输入在展示层有 `true` 兜底 |
| `cost` | 无 | 仅统计，不影响使用 |
| `status` | `active` | 只有 `deprecated` 才会被禁用 |

---

## 5. 最重要的一节：`tool_call` 与 Function Calling 的关系

这是配置自定义模型时**最容易误解**的地方。结论分三层：

### 5.1 `tool_call` 是"能力标记"，不是"开关"

`capabilities.toolcall` 只用于描述/展示模型能力，**不会拦截工具的下发**。真正决定实际请求里 `tools` 数组的，是权限过滤逻辑：

```ts
// session/llm/request.ts:213
return Record.filter(input.tools, (_, k) => input.user.tools?.[k] !== false && !disabled.has(k))
```

它只看 `agent.permission` 和 `user.tools`，**完全不读 `capabilities.toolcall`**。也就是说：

- 即使你把 `tool_call` 设成 `false`，工具仍会被放进请求（除非被权限过滤）。
- `tool_call` 的职责是**标注**，对工具是否发送没有强制作用。

### 5.2 三层"是否支持工具"必须分清

| 层面 | 由什么决定 | 能否靠配置文件解决 |
|------|-----------|------------------|
| opencode 元数据（`capabilities.toolcall`） | `tool_call` 字段，默认 `true` | 没有必要特意配 |
| 实际是否把工具放进请求 | `agent.permission` / `user.tools` | 用权限控制 |
| **服务端是否真支持 function call** | yygu / 网关端点本身 | **只能实测，配置无法保证** |

**核心提醒：** `tool_call: true` 只代表 opencode"允许"发工具，**不代表你的服务端会正确返回 `tool_calls`**。要确认某个模型能不能用 function call，只能向服务端实测。

### 5.3 让一个"能 function call 的模型"真正用起来，需要什么

1. **正确的 `id`** — 实际请求的 `model` 参数 = `api.id`，`id` 缺省时 = models 的键，`name` 仅显示。若服务端要求带前缀的完整名（如 `deepseek/deepseek-v4-flash`），必须显式写 `"id"`，否则请求发错模型名。
2. **OpenAI 兼容的 chat 端点** — function calling 走 chat/completions 的 `tools` + `tool_choice` 结构；无 `npm` 时 opencode 默认按 openai-compatible 处理，方向正确。
3. **权限放行** — 没有 `permission` 把工具 `deny`、`user.tools` 未设 `false`。
4. `tool_call: true` — 可选，语义标注用，默认已是 true。

---

## 6. 如何开启模型的推理（thinking）

> 一句话：`reasoning: true` 只是「**标注**这是推理模型」，**不会自动让请求带上开启推理的参数**。能否真的返回思考内容，取决于网关认哪个请求字段——必须实测，配置无法保证。

### 6.1 三层要分清

| 层面 | 由什么决定 | 能否靠配置 |
|------|-----------|-----------|
| opencode 元数据 `capabilities.reasoning` | `reasoning` 字段，默认 `false` | 写 `true` 只是标注 |
| 请求体里是否带「开启思考」参数 | 各网关规定的字段（如 `enable_thinking`、`thinking`、`reasoning_effort`） | 见 6.4，写在模型 `options` |
| 服务端是否真的返回推理内容 | 网关 / 模型本身 | **只能实测** |

### 6.2 对 openai-compatible 自定义供应商，`reasoning: true` 不会自动加参数

- opencode 只对少数**写死的 providerID** 自动加推理参数（如 `alibaba-cn` 自动 `enable_thinking: true`、`zai`/`zhipuai` 自动 `thinking: {...}`，见 `provider/transform.ts`）。
- 对通用 openai-compatible 自定义供应商，请求 body 顶层是干净的，**不带任何推理字段**（实测抓包确认）。
- deepseek 系唯一自动化的部分是**响应解析**：`interleaved.field = "reasoning_content"`（`provider/provider.ts`），只影响怎么读思考内容，不影响请求侧是否开启。

### 6.3 网关字段实测（yygu / deepseek-v4-flash 实测结果）

| 请求体字段 | 是否返回 `reasoning_content` |
|-----------|------------------------------|
| （无字段，基线） | ❌ |
| `"reasoning_effort": "high"` | ❌ |
| `"thinking": { "type": "enabled", "thought": true }` | ❌ |
| `"enable_thinking": true` | ✅ |

结论：**yygu 网关认 `enable_thinking: true`**。其他网关可能认别的字段（deepseek 官方新版用 `reasoning_effort`、智谱 zhipu 用 `thinking`），务必以你的网关文档 / 实测为准。

### 6.4 正确配置：在模型级 `options` 里写网关认的字段

```jsonc
"models": {
  "deepseek-v4-flash": {
    "name": "deepseek/deepseek-v4-flash",
    "reasoning": true,
    "options": { "enable_thinking": true }
  }
}
```

- `options` 里的键会**原样透传**到 chat/completions 请求 body 顶层（实测抓包确认）。
- 放在**模型级**而非 provider 级，只对该模型生效。
- 改完需重启 opencode。

### 6.5 验证方法

1. **curl 直接对比**：带 / 不带候选字段各打一次端点，看响应里是否含 `reasoning_content`。
2. **本地抓包代理**：临时把 `baseURL` 指向本地转发代理，确认 opencode 发出的 body 顶层确实带上了字段，且响应返回思考内容。

---

## 7. 完整配置模板（可直接复制改用）

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "mygw": {
      "name": "My Gateway",
      "env": ["MYGW_API_KEY"],
      "options": {
        "baseURL": "https://api.example.com/v3",
        "timeout": 120000
      },
      "models": {
        "chat-model": {
          "id": "chat-model",
          "name": "chat-model",
          "family": "general",
          "tool_call": true,
          "limit": { "context": 64000, "output": 8192 },
          "modalities": { "input": ["text"], "output": ["text"] }
        },
        "deepseek-r1": {
          "id": "provider/deepseek-r1",
          "name": "deepseek-r1",
          "reasoning": true,
          "limit": { "context": 128000, "output": 16384 },
          "cost": { "input": 0.5, "output": 1.5 }
        },
        "vision-model": {
          "id": "vision-model",
          "modalities": { "input": ["text", "image"], "output": ["text"] }
        }
      }
    }
  }
}
```

---

## 8. 验证与排查

### 8.1 确认服务端是否支持 function call（唯一可靠手段）

用 curl 带 `tools` 打你的端点，看返回里是否出现 `tool_calls`：

```bash
curl -X POST "https://api.example.com/v3/chat/completions" \
  -H "Authorization: Bearer $MYGW_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "你的模型id",
    "messages": [{"role":"user","content":"用 add 工具计算 1+1"}],
    "tools": [{
      "type": "function",
      "function": {
        "name": "add",
        "description": "加法",
        "parameters": {
          "type": "object",
          "properties": { "a": {"type":"number"}, "b": {"type":"number"} },
          "required": ["a","b"]
        }
      }
    }],
    "tool_choice": "auto"
  }'
```

判读：

- 返回里有 `tool_calls`（含 `add`）→ **服务端支持 function call**。
- 返回报错（如"tools 不被允许"、"tool_choice 不支持"、BadRequest）→ 服务端不支持，改配置无效。

> 端点路径注意：你配置的 `baseURL` 必须是实际的 chat/completions 路径，否则会 404。

### 8.2 查看 opencode 里某个 agent 可用哪些工具

```bash
opencode debug agent <agent名>
```

输出 `tools: { ... }` 是该 agent 解析到的工具及是否 disabled（反映权限层结果；不显示 `toolcall` 标记本身）。

### 8.3 工具不工作的排查顺序

1. **权限**：确认 `opencode.json` / `.opencode` 里没有 `permission` 把工具设成 `deny`，`user.tools` 没有设 `false`。这是最常被忽略的。
2. **服务端不支持**：用 8.1 的 curl 实测，确认端点确实能吃 `tools` 并返回 `tool_calls`。
3. **模型名 / 端点错**：确认 `id` 填的是服务端要求的完整模型名，`baseURL` 是正确路径。

---

## 9. 常见坑清单

- **`id` vs `name`** — 发出去的 model 名取 `id`（缺省 = 键），不是 `name`。只写 `name` 时请求可能发错模型。
- **`limit` 默认 0** — 不配置 context/output 时均按 0 处理，建议显式给。
- **`tool_call` 不控制工具下发** — 它只是能力标记；工具可用性归权限管、服务端能力归实测管。
- **deepseek 系推理** — openai-compatible 且 `id` 含 `deepseek` 时，opencode 自动设 `interleaved.field = "reasoning_content"`（`provider/provider.ts`），无需手配。
- **`reasoning: true` 不开启推理** — 它只是能力标注；请求体是否带「开启思考」参数由网关字段决定，需在模型 `options` 里显式传（如 `enable_thinking: true`）并经实测确认（见「第 6 节」）。
- **无 `npm`** — 默认走 openai-compatible；若你的端点不标准，需要指定正确的 `npm` 包。
- **配置需重启** — opencode 启动时读一次配置，改完要重启才生效。

---

> **关联文档:** [主解读](./README.md) · [系统架构深读](./architecture.md) · [LLM 四轴路由系统](./llm-route-system.md)
