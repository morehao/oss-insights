# Codemap 两阶段生成 — DeepWiki-Open 深入解析

> 所属项目：[DeepWiki-Open](./README.md)
>
> Codemap 是 deepwiki 针对"如何使用 / 怎么改"类问题的分步骤源码引导。本文聚焦 `api/services/codemap.py`，拆解其两阶段 LLM 流程、JSON 自愈解析与引用落点校验。

---

## 一、定位

普通 Wiki 讲"模块是什么、怎么工作"，描述性强但针对性弱；Codemap 则面向**用户的具体使用/改造问题**（"如何添加一个新的 connector？"），把答案组织成**有编号的分节步骤（1, 1a, 1b...）**，且每一步都必须引用真实源码，最终渲染成"带代码查看器联动"的引导页。

输出是一个 `CodeMap` 结构，含 sections → steps → citation（`file_path / start_line / end_line / snippet`）。

---

## 二、两阶段 LLM 流（`api/services/codemap.py:225`）

为控制成本与成功率，不一次让 LLM 输出完整 codemap，而是拆成两个语义不同的阶段：

```
Phase 1a: analyzing       RAG 检索 → 收集 relevant chunks，按文件分组
Phase 1b: initial_codemap  skeleton 调用 → 只出结构（sections/steps/citations）
Phase 2:  diagrams        enrich 调用 → 给每节补 prose guide + Mermaid diagram
```

```python
# Phase 1b — 先出骨架
skeleton = CodeMap.model_validate(
    await _generate_json(_new_streamer, skeleton_prompt)
)
yield _phase("initial_codemap", "done", section_count=len(skeleton.sections))

# Phase 2 — 再富化（失败可降级为骨架）
final = skeleton
try:
    final = CodeMap.model_validate(
        await _generate_json(_new_streamer, enrich_prompt, attempts=2)
    )
except Exception:
    yield _phase("diagrams", "done", degraded=True)   # 非致命，回退
```

好处：骨架阶段 prompt 更短、更稳；富化阶段即使失败（本地小模型常栽在 Mermaid 语法），**至少还能给用户一个可用的纯骨架**。

---

## 三、JSON 自愈解析（`_extract_json` + `_generate_json`）

LLM 输出 JSON 很不稳定，项目做了两层防御：

### 1. 单次输出的宽容解析（`codemap.py:77`）

- 剥掉 ` ```json / ``` ` 代码围栏；
- 用**括号深度扫描**定位第一个平衡的顶层对象（忽略字符串内的花括号）；
- 解析失败再走 `_repair_json` 修复常见污染（`codemap.py:69`）：
  - 尾部逗号 `,\s*([}\]])` → `\1`
  - 多余空格导致 `" "key":` → `"key":`

### 2. 全局重试（`_generate_json`，`codemap.py:46`）

```python
async def _generate_json(streamer_factory, prompt, attempts=3):
    for attempt in range(1, attempts + 1):
        raw = await _collect_stream(streamer_factory(), prompt)
        try:
            return _extract_json(raw)
        except Exception as e:
            logger.warning("JSON parse attempt %d/%d failed: %s", ...)
    raise ValueError(f"Model did not return valid JSON after {attempts} attempts: ...")
```

关键细节：**每次重试用全新的 `streamer_factory()`**——对小模型，同一次生成可能始终带同样毛病，重开连接 + 重发提示让随机性重新洗牌，提高命中率。

---

## 四、引用落点校正（Grounding Citations）

这是对抗幻觉最巧妙的一环。LLM 回填的 `start_line / end_line` 是**猜的**，但 `snippet` 是从上下文原样复制（prompt 强约束"copy verbatim"）。于是：

```python
def _ground_citations(codemap, repo_dir):
    ...
    loc = _locate_snippet(text, cit.snippet)   # 在真实源码里子串定位
    if loc:
        cit.start_line, cit.end_line = loc
```

思路：**与其信 LLM 的行号，不如把 snippet 当"指纹"去真实文件里搜索**（`_locate_snippet`，`codemap.py:178`；先精确整段搜索，失败退化为首行锚点）。这样一个文件的真实行号被精确覆盖，前端代码查看器就能高亮到对的代码。

### 配套的安全设计

`read_repo_file`（`codemap.py:166`）用于读取文件供给前端查看，包含**路径穿越防护**：

```python
repo_dir = os.path.realpath(local_repo_dir(repo_url, repo_type))
target = os.path.realpath(os.path.join(repo_dir, file_path))
if os.path.commonpath([repo_dir, target]) != repo_dir:
    raise ValueError("Resolved path escapes the repository directory")
```

用 `realpath` 消除 `../` 与符号链接后，用 `commonpath` 确保目标仍在仓库根内，否则拒绝——避免用户通过 `/codemap/file?file_path=../../etc/passwd` 读取任意文件。

---

## 五、NDJSON 事件流 + 前端联动

生成结果通过 NDJSON 事件逐条推送（`codemap.py:29` 序列化每个事件）：

| 事件类型 | 载荷 | 语义 |
|----------|------|------|
| `phase` | name/status(+counts) | 三阶段进度：analyzing → initial_codemap → diagrams |
| `codemap` | 完整 `CodeMap` dump | 最终结果 |
| `error` | stage/message | 骨架阶段失败（致命） |
| `done` | — | 结束标记 |

路由 `/ws/codemap`（WebSocket）为主、`/codemap/stream`（NDJSON over HTTP）为兜底。前端 `CodeViewer` 组件注册为模态级抽屉：点击某步骤的 citation → 通过 `/codemap/file?repo_url=&file_path=` 拉取该文件全文 → 用校正后的 `start_line/end_line` 高亮对应行。

---

### 关键文件索引

| 文件 | 职责 |
|------|------|
| `api/services/codemap.py` | 阶段编排、JSON 自愈解析、引用落点、路径防护 |
| `api/prompts.py` | `CODEMAP_SKELETON_PROMPT` / `CODEMAP_ENRICH_PROMPT` |
| `api/schemas/codemap.py` | `CodeMap` / `CodeMapRequest` 模型 |
| `api/routers/codemap.py` | WebSocket + HTTP + 文件读取路由 |
| `src/components/CodeViewer.tsx` | 前端代码查看 / 行高亮联动 |
