# 多 Provider 流式架构 — DeepWiki-Open 深入解析

> 所属项目：[DeepWiki-Open](./README.md)
>
> 本文聚焦 `api/chat/`、`api/clients/` 与各 router，分析如何把 9 个 LLM Provider 统一为一个流式抽象，以及 WebSocket / SSE 双通道的降级设计。

---

## 一、问题

DeepWiki 要支持从云端（Google/OpenAI/Anthropic）到本地离线（Ollama）的连续谱系模型后端。若每个端点在 `research_chat` 里用 if-else 分发，新增 Provider 就要改核心逻辑，且各家的流式协议差异极大（OpenAI 是 chunk 增量、Anthropic 是 content_block_delta、Ollama 是原生流）。需要一个统一抽象。

---

## 二、`ChatStreamer` — 工厂 + 注册表

核心抽象在 `api/chat/_stream.py:29`：

```python
class ChatStreamer(ABC):
    _registry: dict[str, type["ChatStreamer"]] = {}
    provider: str
    error_hint: str | None = None

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        if provider := getattr(cls, "provider", None):
            ChatStreamer._registry[provider] = cls

    @classmethod
    def create(cls, *, provider, model=None, model_config):
        registered = ChatStreamer._registry.get(provider, None)
        if registered:
            return registered(model=model, model_config=model_config)
        raise RuntimeError(f"Provider {provider} not registered")

    @abstractmethod
    def respond_stream(self, prompt: str) -> AsyncIterator[str]: ...
```

设计要点：

- **声明式注册**：每个子类声明 `provider = "ollama"` 等，`__init_subclass__` 自动进注册表。核心代码永远不出现 Provider 名字——新增只需加一个类。
- **只暴露一个接口** `respond_stream → AsyncIterator[str]`：所有对话（简单 chat / Deep Research / Wiki 页生成 / Codemap）都只需消费字符串流，无关底层协议。

### Provider 家族

| 类 | provider | 说明 |
|----|----------|------|
| `OllamaChatStreamer` | ollama | Adal 的 OllamaClient，过滤 `<think>` 标签 |
| `OpenRouterChatStreamer` | openrouter | 直连 OpenRouterClient |
| `OpenAIChatStreamer` | openai | Linux 兼容（`_OpenAICompatStreamer` 基类） |
| `AzureChatStreamer` | azure | 同 OpenAI 兼容基类 |
| `LiteLLMChatStreamer` | litellm | 通过 LiteLLM 路由到更多后端 |
| `BedrockChatStreamer` | bedrock | 一次返回整串（非逐 chunk） |
| `DashScopeChatStreamer` | dashscope | 阿里云灵积 |
| `GoogleGenerativeChatStreamer` | google | `google.generativeai` 异步流 |
| `AnthropicChatStreamer` | anthropic | Bedrock 上的 Claude，解析 `content_block_delta` |

### 复用基类 `_OpenAICompatStreamer`

`_OpenAICompatStreamer`（`_stream.py:137`）抽象 OpenAI 兼容的公共逻辑：构造 `model_kwargs`（含 `temperature`、条件 `top_p`）、解析 `choices[0].delta.content`。OpenAI / Azure / LiteLLM 三个子类只须实现 `_build_client()` 返回各自的 `OpenAIClient`：

```python
class AzureChatStreamer(_OpenAICompatStreamer):
    provider = "azure"
    def _build_client(self):
        from api.clients import AzureAIClient
        return AzureAIClient()
```

---

## 三、`ModelClient` 层（`api/clients/`）

`ChatStreamer` 之上还有一层 Adal 的 `ModelClient` 抽象。每个 Provider 一个文件，统一实现：

- `convert_inputs_to_api_kwargs(input, model_kwargs, model_type)` — 把通用输入转成各 API 的调用参数（`api/clients/anthropic.py:115`）。
- `call` / `acall`（同步/异步）— 真正的网络调用。
- 序列化 `to_dict` / `from_dict` — 支持 config 反序列化。

Anthropic 客户端还内建了 `backoff.expo` 重试（`anthropic.py:134`），对超时 / 限流 / 5xx 指数退避，避免瞬时抖动让长任务失败。

`api/config.py` 的 `CLIENT_CLASSES` + `_DEFAULT_PROVIDER_MAP`（`config.py:71-95`）把请求里的 `provider` 从 JSON 配置解析成具体类，形成 **config → class** 的完整链路。

---

## 四、流式降级（`api/services/research.py:282`）

统一 `AsyncIterator[str]` 让降级逻辑可以集中写一次：

```python
async def stream_and_fallback(streamer, prompt_func, simplified_prompt_func):
    try:
        async for chunk in streamer.respond_stream(prompt_func()):
            yield chunk
    except Exception as e:
        if is_token_limit_error(e):
            # 降级①：去掉上下文重试
            async for chunk in streamer.respond_stream(simplified_prompt_func()):
                yield chunk
            # 仍失败 → 友好错误
            yield "...请求过大..."
        else:
            # 非 token 错误 → 附上 error_hint
            yield f"\nError with {streamer.provider} API: {e}"
            if streamer.error_hint:
                yield f"\n\n{streamer.error_hint}"
```

`error_hint`（如 `"Please check ... OPENAI_API_KEY ..."`）把"缺 key"这类常见问题直接透传给前端用户，避免黑盒堆栈。

---

## 五、双通道传输（WebSocket + SSE）

前端为流式结果准备了**双通道 + 降级**：

### 后端

| 通道 | 端点 | 格式 | 用途 |
|------|------|------|------|
| WebSocket | `/ws/chat` | 纯文本增量 | Chat / 页面生成 / 结构生成 |
| HTTP SSE | `/chat/completions/stream` | `text/event-stream` | 兜底 |

`api/routers/chat.py:20` 的 WebSocket 端点做请求校验后 `async for chunk in await research_chat(request)` 逐条 `send_text`。`finally` 里统一 close（`_send_if_connect` 防止半开连接报错）。

### 前端

`page.tsx` 中（`generatePageContent:704`）先尝试 WebSocket：

1. **5 秒连接超时** → 降级 HTTP；
2. `ws.onerror` → 降级 HTTP；
3. HTTP 用 `ReadableStream` + `TextDecoder` 增量读 body。

```ts
const ws = new WebSocket(wsUrl);
await new Promise((resolve, reject) => {
  const timeout = setTimeout(() => reject(new Error('WebSocket connection timeout')), 5000);
  ws.onopen = () => { clearTimeout(timeout); ws.send(JSON.stringify(requestBody)); resolve(); };
  ws.onerror = () => reject(new Error('WebSocket connection failed'));
});
```

这种"WS 失败自动切 SSE"的再保险，让生成长任务在连接不稳时也能拿到完整结果。

---

## 六、长任务心跳（`api/routers/repo.py`）

`/repo/prepare`（索引构建）是纯 SSE 长任务，最怕反代（nginx / undici）超过 bodyTimeout 掐断。解法：

- 立刻首字节 `": indexing-start\n\n"` 刷新响应头，解除代理超时；
- 每 10s 心跳 `event: progress`；
- 用 `asyncio.shield(task)` + `wait_for(timeout=10)`：**心跳超时绝不能取消正在跑的索引任务**（`repo.py:40-49`）。

```python
while not task.done():
    try:
        await asyncio.wait_for(asyncio.shield(task), timeout=_HEARTBEAT_INTERVAL_SEC)
    except asyncio.TimeoutError:
        yield f"event: progress\ndata: {{'elapsed_sec': elapsed}}\n\n"
```

---

### 关键文件索引

| 文件 | 职责 |
|------|------|
| `api/chat/_stream.py` | `ChatStreamer` 抽象、注册表、9 个 Provider 实现 |
| `api/clients/anthropic.py` | `ModelClient` 示例 + backoff 重试 |
| `api/config.py` | provider 配置解析、Client 类映射 |
| `api/services/research.py` | prompt 组装、流式降级 |
| `api/routers/chat.py` / `repo.py` | WebSocket、SSE、心跳 |
