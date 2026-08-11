# RAG 与 Deep Research 管线 — DeepWiki-Open 深入解析

> 所属项目：[DeepWiki-Open](./README.md)
>
> 本文聚焦 `api/rag/`、`api/services/research.py` 的实现，讲解一个仓库如何被向量化、索引并用于多轮深挖式问答。

---

## 一、整体数据流

仓库从 URL/本地路径到可回答问题的完整管线可拆为四个阶段：

```
Repo 下载/本地路径
      │
      ▼ ① read_all_documents()
按 code/doc 扩展名 + include/exclude 规则读取文件
      │  每文件校验 token 数（>81920 跳过）
      ▼ ② transform_documents_and_save_to_db()
顺序执行 [文本切分, 向量化] → LocalDB 持久化为 pkl
      │
      ▼ ③ FAISS 检索
RAG.prepare_retriever() 构建 FAISS，按 query 取 top-k
      │
      ▼ ④ 组装上下文
按文件分组 + 行号标注 → 拼进 prompt → ChatStreamer 流式问答
```

---

## 二、文档读取与过滤（`api/rag/pipeline.py`）

### 双模式过滤

`read_all_documents`（`pipeline.py:151`）支持两种互斥的过滤策略：

- **Inclusion 模式**：传了 `included_dirs` / `included_files` 时启用，只处理指定目录/文件。
- **Exclusion 模式**（默认）：合并 `repo.json` 的默认排除集 + 请求方额外排除集。

判定逻辑集中在 `_should_process_file`（`pipeline.py:72`）：目录成员判断用 `clean_included in file_path_parts`（即路径中某一层命中即可），文件则做后缀/文件名匹配。

> **实现细节**：目录属于**集合成员判断**而非"前缀匹配"，因此用户填 `src/app` 时，任意含 `src/app` 段的路径都算命中——灵活但可能引入过宽匹配。

### 元数据标注

每个 `Document` 携带丰富的 `meta_data`：

```python
doc = Document(
    text=content,
    meta_data={
        "file_path": relative_path,
        "type": file_ext,
        "is_code": is_code,
        "is_implementation": is_implementation,
        "title": relative_path,
        "token_count": token_count,
    },
)
```
`is_implementation` 靠启发式（路径不含 `test_` / `app_` / `test`）区分实现文件——后续检索权重逻辑可据此倾斜。

---

## 三、切分与向量化（Adal 编排）

DeepWiki 直接复用 **AdalFlow** 的声明式组件，而非常见的手写循环：

```python
data_transformer = adal.Sequential(
    splitter, embedder_transformer
)  # pipeline.py:366
```

`LocalDB` 注册 transformer 后 `.load(documents)` + `.transform()` 一气呵成，再 `save_state` 落盘（`pipeline.py:392-399`）。

### 亮点：`LineTrackingTextSplitter`

Adal 的 `TextSplitter` 会把父文档的 `meta_data` **深拷贝一次后共享给每个 chunk**（节省内存）。但这导致无法给每个 chunk 记独立行号。DeepWiki 用一个子类绕开该限制（`pipeline.py:298`）：

```python
def call(self, documents):
    ...
    for parent_id, chunks in chunks_by_parent.items():
        text = parent_text.get(parent_id, "")
        cursor = 0
        for chunk in sorted(chunks, key=lambda c: c.order):
            chunk.meta_data = dict(chunk.meta_data or {})   # 每个 chunk 独立 copy
            pos = text.find(chunk.text, cursor)             # 保序子串定位
            chunk.meta_data["start_line"] = text.count("\n", 0, pos) + 1
            chunk.meta_data["end_line"] = start_line + chunk.text.count("\n")
            cursor = pos + 1
```

关键技巧：**用保序的 `text.find(chunk.text, cursor)`** 计算每个 chunk 的真实行范围。这为前端"引用带行号"提供了数据源，Codemap 与 Wiki 页都能把检索结果精确映射到源码行。

### Embedding 异构兜底

`prepare_db_index` 加载旧索引时先统计各向量维度：
- 全为 0 → 视为废索引，重建；
- 维度不一致 → `_validate_and_filter_embeddings`（`api/rag/rag.py:184`）保留**众数维度**、丢弃离群，避免 FAISS 构建失败。

---

## 四、检索与问答（`api/rag/rag.py`）

`RAG` 是一个 `adal.Component`，构造时初始化 `Memory`（`CustomConversation`，`rag.py:75`，修复了原库的列表索引越界问题）与 embedder。

### 异步非阻塞

```python
async def aprepare_retriever(self, ...):
    async with _get_rag_semaphore():   # 并发上限 DEEPWIKI_MAX_CONCURRENT_RAG=4
        return await asyncio.to_thread(self.prepare_retriever, ...)
```

所有阻塞操作（clone/IO/embedding）扔到工作线程，并用信号量限流，避免多个请求同时触发昂贵的索引构建打崩事件循环。

### 组装上下文

`research_chat`（`api/services/research.py:59`）把检索到的 chunk **按文件路径分组**，并在每个 chunk 前标注 `[lines 起始-结束]`：

```
## File Path: src/app/api/chat/stream/route.ts
[lines 12-30]
export const POST = ... 
```

这样 LLM 既能读到代码，又能精确引用行号返回给前端做进链接。

---

## 五、Deep Research：多轮迭代问答

除了单轮问答，项目实现了 **Deep Research（深挖）** 模式（`research_chat:117-254`）：

| 迭代号 | prompt | 目的 |
|--------|--------|------|
| 任意 | 首条消息检测"continue ... research"则替换为原始主题（`research.py:124`） | 允许一个 topic 跨多轮延续 |
| 1 | `DEEP_RESEARCH_FIRST_ITERATION_PROMPT` — 输出 `## Research Plan` + `## Next Steps` | 建立研究计划 |
| 2-4 | `...INTERMEDIATE...` — 输出 `## Research Update {n}` | 逐轮深挖单一层面 |
| ≥5 | `...FINAL...` — 输出 `## Final Conclusion` | 收敛总结 |

核心机制是**把"研究主题"与"研究回合"作为显式字段**：`ChatMessage.mode: Literal["normal","deep_research"]` + `research_iteration`（`schemas/chat.py`），比用文本标记（旧版）更结构可控。

对话历史通过 `rag.memory.add_dialog_turn()` 逐对加入，最终以 `<turn>` 块拼进 prompt（`research.py:274`）。

---

### 关键文件索引

| 文件 | 职责 |
|------|------|
| `api/rag/pipeline.py` | 文档读取/过滤、切分、向量化、pkl 持久化 |
| `api/rag/rag.py` | RAG 组件、FAISS 检索、embedding 校验、异步封装 |
| `api/services/research.py` | prompt 组装、Deep Research 迭代、流式降级 |
| `api/schemas/chat.py` | 消息与请求模型（mode / research_iteration） |
| `api/config.py` | embedder 类型选择、`get_model_config` |
