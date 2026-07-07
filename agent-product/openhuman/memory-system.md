# OpenHuman 记忆系统深度分析

---

## 一、系统概述

OpenHuman 的记忆系统是一个**分层、模块化、可插拔**的持久化与检索架构。它不是简单的向量数据库封装，而是一个完整的记忆生命周期管理系统——从原始数据摄入、分块、向量化、树状摘要、知识图谱、实体提取，到混合检索和变更追踪。

**核心设计哲学**：记忆是 Agent 的"长期上下文"，而非简单的存储后端。系统围绕"摄取→索引→摘要→检索→遗忘"全链路设计。

---

## 二、整体架构

```
React Frontend (MemoryWorkspace, MemoryGraph, MemoryInsights)
         │ JSON-RPC (openhuman.memory_*)
         ▼
  ┌─────────────────────────────────┐
  │  memory (orchestrator)          │ ← 编排层，对外暴露 RPC
  │  sync / remember / ingest       │
  │  query (fast_walk / drill_down) │
  │  tools (recall / store / forget)│
  └──────┬──────────┬───────────────┘
         │          │
         ▼          ▼
  ┌──────────┐ ┌──────────┐ ┌──────────────┐
  │mem_store │ │mem_tree  │ │ mem_sync     │
  │(持久化)   │ │(摘要树)   │ │(同步管线)     │
  ├──────────┤ ├──────────┤ ├──────────────┤
  │SQLite   │ │桶密封     │ │Composio:     │
  │.md 文件  │ │摘要生成   │ │ Gmail/Slack  │
  │向量存储   │ │检索遍历   │ │ GitHub/Notion│
  │KV 存储   │ │嵌入评分   │ │Workspace观察 │
  └────┬─────┘ └────┬─────┘ │MCP 服务器     │
       │             │       └──────────────┘
       ▼             ▼
  ┌──────────────────────────────────────┐
  │          On-disk Storage             │
  │ SQLite (WAL) + .md files + JSONL    │
  │ + Git-based diff ledger              │
  └──────────────────────────────────────┘
```

---

## 三、核心模块详解

### 3.1 memory（编排层）

`src/openhuman/memory/` — 系统的入口点和协调者。

| 子模块 | 职责 |
|--------|------|
| `sync.rs` | 同步调度：触发同步管线，管理同步状态 |
| `remember.rs` | 记忆构建：文档摄取入口 |
| `ingest_pipeline.rs` | 摄取管线编排：解析→丰富→持久化→图谱 |
| `chat.rs` | 对话记忆管理 |
| `query/` | 查询策略：`fast_walk`（智能遍历）、`drill_down`（确定性遍历）、`cover_window`（上下文组装）、`search_entities`（实体索引） |
| `ops/` | 业务逻辑：learn、sync、kv_graph、files、documents、tool_memory |
| `read_rpc/` | 只读 RPC：admin、vault、entities、chunks、graph |
| `schema/` | RPC 控制器注册 |
| `tools/` | Agent 工具：`recall`、`store`、`forget`、`doctor` |
| `preferences.rs` | 用户偏好：`general`（始终注入提示）+ `situational`（按轮语义检索） |
| `ingestion/` | 摄取队列、状态机、解析规则、正则提取 |

**关键设计**：编排层不直接操作存储，而是通过 `MemoryClient`（线程安全 `Arc` 包装）委派给 `memory_store`。

### 3.2 memory_store（持久化层）

`src/openhuman/memory_store/` — 所有数据的真实来源。

#### 核心结构体

**`UnifiedMemory`**：一个 SQLite 连接（WAL 模式）+ 磁盘 `.md` 侧边目录 + 向量存储路径。字段：
- `workspace_dir` — 工作区根目录
- `memory_dir` — 记忆目录
- `db_path` — SQLite 数据库路径
- `vectors_dir` — 向量存储路径
- `conn` — SQLite 连接
- `embedder` — 嵌入器实例

**`MemoryClient`**：客户端门面，通过 `MemoryClientRef = Arc<MemoryClient>` 线程安全访问。主要方法：`put_doc`、`put_doc_light`、`ingest_doc`、`list_documents`、`list_namespaces`、`query_namespace`、`recall_namespace`、`graph_query`、`kv_set/get/delete`。

#### 存储子模块

| 子模块 | 文件 | 职责 |
|--------|------|------|
| **chunks/** | `types.rs`, `store.rs`, `produce.rs`, `semantic.rs`, `embeddings.rs` | 块完整生命周期 |
| **content/** | `raw.rs`, `atomic.rs`, `obsidian.rs`, `compose/`, `paths.rs`, `tags.rs`, `wiki_git/` | 真实来源：每个正文对应一个 `.md` 文件，不可变正文 |
| **trees/** | `store.rs`, `types.rs`, `registry.rs`, `hotness.rs` | 树持久化 |
| **vectors/** | `store.rs` | 本地向量数据库，余弦相似度，按签名分桶的每模型向量 |
| **entities.rs** | (从 memory_tree 重导出) | 实体索引 |
| **kv.rs** | | 键值存储 |
| **unified/** | `documents.rs`, `query.rs`, `segments.rs`, `events.rs`, `profile.rs` | 统一查询表面（逐渐弃用） |

#### Memory 核心 Trait

```rust
trait Memory {
    fn store(..) -> Result<()>;           // 写入条目的 provenance 数据
    fn store_with_taint(..) -> Result<()>;
    fn recall(..) -> Result<Vec<Entry>>;  // 关键词/语义检索
    fn recall_relevant_by_vector(..);      // 纯向量检索（阈值门控）
    fn get(..) / list(..) / forget(..);    // CRUD
    fn namespace_summaries(..) / count(..) / health_check(..);
    fn sqlite_conn() -> &Connection;       // 子系统逃生舱口
}
```

#### 核心值类型

- **`MemoryEntry`**：`{ id, key, content, namespace, category, timestamp, session_id, score, taint }`
- **`MemoryCategory`**：`Core | Daily | Conversation | Custom(String)`
- **`MemoryTaint`**：`Internal | ExternalSync` — 安全关键的 provenance，失败时默认 `ExternalSync`
- **`MemoryKind`**：`Raw | Chunk | Entity | Tree | Vector | Kv | Contact`
- **`VectorEmbeddable`** / **`ObsidianRepresentable`**：每种持久化类型必须实现的特质

#### Chunk 元数据

```rust
Chunk { id, content, metadata, seq_in_source, token_count }
Metadata { source_kind, source_id, timestamp, owner, tags, path_scope }
SourceKind { Chat, Email, Document, WebPage, Rss, Github }
```

---

### 3.3 memory_tree（摘要树引擎）

`src/openhuman/memory_tree/` — 通用摘要树引擎，是整个记忆系统最具创新性的组件。

#### 桶密封算法

```
叶子追加 → L0 缓冲区
    ↓ 当 SUMMARY_FANOUT 溢出
桶密封：LLM 摘要 L0 → 成为 L1 叶子
    ↓ 当 L1 再次溢出
桶密封：LLM 摘要 L1 → 成为 L2 叶子
    ↓ 循环直到稳定
```

#### 关键类型

- **`TreeWriteRequest`**：`{ tree_id, tree_kind, leaf, label_strategy, deferred }`
- **`TreeWriteOutcome`**：`{ new_summary_ids, seal_pending }`
- **`TreeReadRequest`**：`{ tree_id, start_node_id, max_depth, query, limit }`
- **`TreeReadHit`**：`{ node_id, node_kind, level, content, score }`
- **`TreeLeafPayload`**：`{ chunk_id, token_count, timestamp, content, entities, topics, score }`
- **`SummaryNode`**：`{ id, tree_id, level, content, token_count, entities, topics, score, sealed_at }`
- **`Buffer`**：`{ item_ids, token_sum, oldest_at }`
- **`Tree`**：`{ id, kind, scope, root_id, max_level, status }`

#### 检索策略

| 策略 | 文件 | 说明 |
|------|------|------|
| **walk** | `retrieval/walk.rs` | 智能遍历：Agent 探索摘要节点，逐层深入 |
| **drill_down** | `retrieval/drill_down.rs` | 确定性遍历：从已知摘要精确下钻 |
| **fetch_leaves** | `retrieval/fetch_leaves.rs` | 直接拉取原始叶子 |
| **query_source** | `retrieval/query_source.rs` | 源范围检索 |
| **search_entities** | `retrieval/search_entities.rs` | 实体索引查找 |
| **cover_window** | `retrieval/cover.rs` | 上下文组装窗口 |

#### 评分系统 (`memory_tree/score/`)

| 子模块 | 说明 |
|--------|------|
| **signals/** | `source_weight`, `metadata_weight`, `token_count`, `unique_words`, `interaction` |
| **embed/** | 嵌入器工厂：`InertEmbedder`, `OpenAiCompatEmbedder`, `OllamaEmbedder`, `CloudEmbedder` |
| **extract/** | 实体提取器：`RegexEntityExtractor`（确定性），`LlmEntityExtractor`（LLM 驱动），`resolver` |

#### 摘要生成

`summarise.rs` — 单一函数，使用 LLM 产生 `L_n → L_{n+1}` 摘要正文。

---

### 3.4 memory_sync（同步管线）

`src/openhuman/memory_sync/` — 外部数据源同步框架。

#### SyncPipeline Trait

```rust
trait SyncPipeline {
    fn id() -> &'static str;
    fn kind() -> SyncKind;           // Composio | Workspace | Mcp
    fn init(config) -> Result<()>;
    fn tick(config) -> SyncOutcome { records_ingested, more_pending, note }
}
```

#### 管线实现

| 管线 | 来源 | 实现路径 |
|------|------|----------|
| **Composio** | Gmail, Slack, GitHub, Notion, Linear, ClickUp | `memory_sync/composio/` — 分提供者实现，鼓风机，连接器 |
| **Workspace** | 本地文件 Vault | `memory_sync/workspace/` — 使用 `notify` crate 的文件观察器 |
| **MCP** | 第三方 MCP 服务器 | `memory_sync/mcp/` |

#### 规范化 (`memory_sync/canonicalize/`)

将上游数据转换为规范 `.md` 格式：
- `chat.rs` — 聊天消息
- `email.rs` — 邮件
- `document.rs` — 文档
- `email_clean.rs` — 邮件清理

---

### 3.5 摄取管线

`src/openhuman/memory/ingestion/` — 采用发布-订阅模式，四步流水线：

```
1. 解析 (Parse)
   原始文本 → ParsedIngestion { chunks, entities, relations }
   
2. 丰富 (Enrich)
   添加元数据、标签、路径范围
   
3. 持久化 (Persist)
   upsert_document() → SQLite + .md 文件
   
4. 图谱 (Graph)
   upsert_graph_relations() → 知识图谱
```

#### 默认参数

| 参数 | 默认值 |
|------|--------|
| `DEFAULT_CHUNK_TOKENS` | 225 |
| `DEFAULT_MEMORY_EXTRACTION_MODEL` | `"heuristic-only"` |
| 实体阈值 | 0.45 |
| 关系阈值 | 0.30 |
| 相邻阈值 | 0.50 |
| 提取模式 | `ExtractionMode::Sentence`（默认） |

#### 语义关系规则

20 种验证谓词，每种有严格的主语/宾语类型约束：
`OWNS`, `WORKS_ON`, `RESPONSIBLE_FOR`, `USES`, `WORKS_FOR`, `DEPENDS_ON`, `PREFERS`, `HAS_DEADLINE`, `COMMUNICATES_WITH`, `INVESTIGATES`, `NORTH_OF`, `SOUTH_OF`, `EAST_OF`, `WEST_OF`, `AVOIDS`, `LED_BY`, `HAS_MEMBER`, `ASSIGNED_TO`, `ATTENDED`, `CREATED`, `ORGANIZED_BY`

---

### 3.6 其他记忆模块

#### memory_conversations
- 基于 JSONL 的对话线程/消息存储
- 路径：`<workspace>/memory/conversations/`

#### memory_diff（变更追踪）
- 基于 Git 的快照式变更检测
- 快照存储为 Git commit
- 检查点存储为 Git tag
- 阅读标记存储为 Git ref

#### memory_entities（实体注册表）
- 基于 Obsidian 格式的实体存储
- 15 种实体类型：从 Person 到 Misc

#### memory_graph（知识图谱）
- 从实体索引派生的共现边
- Graph query RPC

#### memory_goals（目标系统）
- 基于目标的实体丰富 RPC

#### memory_sources（数据源注册表）
- 连接器注册：文件夹、GitHub、RSS、Web

#### memory_queue（摄取队列）
- 第三方队列实现，使用 memory_store 作为后端

---

## 四、事件系统

所有记忆事件挂载在 `"memory"` 域下（`DomainEvent` 枚举，定义于 `src/core/event_bus/events.rs`，共 1629 行）：

| 事件 | 触发时机 |
|------|----------|
| `MemoryStored` | 条目存入 |
| `MemoryRecalled` | 检索完成 |
| `MemorySyncRequested` | 同步请求 |
| `MemorySyncStageChanged` | 同步阶段变更 |
| `MemoryIngestionStarted` | 摄取开始 |
| `MemoryIngestionCompleted` | 摄取完成 |
| `MemoryDiffSnapshotTaken` | 快照拍摄 |
| `MemoryDiffComputed` | 变更计算 |
| `MemoryDiffMarkedRead` | 标记已读 |
| `MemoryTreeBuildProgress` | 树构建进度 |
| `EmbeddingModelUnhealthy` | 嵌入模型不健康 |
| `ProviderApiKeyRejected` | API Key 被拒 |

---

## 五、配置系统

### MemoryConfig

定义于 `src/openhuman/config/schema/storage_memory.rs`：

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `backend` | `"sqlite"` | 存储后端 |
| `auto_save` | `true` | 自动保存 |
| `embedding_provider` | `"cloud"` | 嵌入提供者：`cloud`, `ollama`, `openai`, `cohere`, `voyage`, `custom:<url>` |
| `embedding_model` | `"embedding-v1"` | 嵌入模型 |
| `embedding_dimensions` | `1024` | 嵌入维度 |
| `embedding_rate_limit_per_min` | `60` | 速率限制 |
| `min_relevance_score` | `0.4` | 最小相关分数 |

### MemoryTreeConfig

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `embedding_endpoint` | 无 | Ollama 嵌入端点 |
| `embedding_model` | `"bge-m3"` | Ollama 模型 |
| `embedding_strict` | `true` | 无嵌入器时保错/降级 |
| `llm_extractor_endpoint` | `http://localhost:11434` | 实体提取 LLM |
| `llm_extractor_model` | `"gemma3:4b"` | 实体提取模型 |
| `llm_summariser_endpoint` | `http://localhost:11434` | 摘要 LLM |
| `llm_summariser_model` | `"gemma3:4b"` | 摘要模型 |
| `llm_summariser_timeout` | `120s` | 摘要超时 |
| `llm_backend` | `Cloud` | 摘要后端 |
| `cloud_summarization_opt_in` | `false` | 云端摘要必须选择加入 |
| `spacy_enabled` | `true` | spaCy NER sidecar |

### 嵌入默认值

| 常量 | 值 | 来源 |
|------|-----|------|
| `DEFAULT_CLOUD_EMBEDDING_MODEL` | `"embedding-v1"` (voyage-3.5) | `embeddings/cloud.rs` |
| `DEFAULT_CLOUD_EMBEDDING_DIMENSIONS` | 1024 | `embeddings/cloud.rs` |
| `DEFAULT_OLLAMA_MODEL` | `"bge-m3"` | `embeddings/ollama.rs` |
| `DEFAULT_OLLAMA_DIMENSIONS` | 768 | `embeddings/ollama.rs` |

---

## 六、用户偏好系统

`memory/preferences.rs` — 两级用户偏好：

| 级别 | 场景 | 阈值 |
|------|------|------|
| **general** | 始终注入系统提示 | 无阈值 |
| **situational** | 按轮语义检索 | `SITUATIONAL_MIN_SIMILARITY = 0.35` |
| **contradiction** | 检测矛盾 | `CONTRADICTION_SIMILARITY = 0.6` |

---

## 七、实体提取

两层方法：

| 层 | 提取器 | 特性 |
|----|--------|------|
| **L1** | `RegexEntityExtractor` | 确定性，始终运行，基于规则 |
| **L2** | `LlmEntityExtractor` | 可选，默认使用 `gemma3:4b`，深度语义提取 |

15 种实体类型：Person 到 Misc。

---

## 八、前端

### Tauri 命令 (`app/src/utils/tauriCommands/memory.ts`)

通过 `core_rpc_relay` 调用 Rust 核心的 `openhuman.memory_*` 方法：
- `memoryListDocuments`、`memoryListNamespaces`、`memoryDeleteDocument`
- `memoryQueryNamespace`、`memoryRecallNamespace`、`memoryGraphQuery`
- `memoryDocIngest`、`memorySyncChannel`、`memorySyncAll`
- `aiListMemoryFiles`、`aiReadMemoryFile`、`aiWriteMemoryFile`

### React 组件 (`app/src/components/intelligence/`)

| 组件 | 功能 |
|------|------|
| `MemoryWorkspace` | 记忆工作区主面板 |
| `MemoryNavigator` | 记忆导航器 |
| `MemoryGraph` | 知识图谱可视化 |
| `MemoryTimelinePanel` | 时间线面板 |
| `MemoryFreshnessPanel` | 数据新鲜度面板 |
| `MemoryStatsBar` | 统计状态栏 |
| `MemoryInsights` | 洞察面板 |
| `MemoryChunkDetail` / `MemoryChunkScoreBars` / `MemoryChunkMentioned` | Chunk 详情 |
| `MemorySourceRegistry` / `AddMemorySourceDialog` | 数据源注册 |
| `MemoryHeatmap` / `MemoryControls` / `MemoryResultList` | 控制与结果 |
| `IntelligenceMemoryTab` / `MemorySection` / `MemoryEmptyPlaceholder` | 布局组件 |
| `MemoryDataPanel` / `MemorySyncPanel` / `MemoryDebugPanel` / `MemoryWindowControl` | 设置面板 |

---

## 九、测试覆盖

23 个端到端测试文件 (`tests/`)：
- `memory_fast_retrieve_e2e.rs`
- `memory_sync_pipeline_e2e.rs`
- `memory_tree_sync_raw_coverage_e2e.rs`
- `memory_sources_e2e.rs`
- `memory_raw_coverage_e2e.rs`
- `memory_graph_sync_e2e.rs`
- `memory_roundtrip_e2e.rs`
- `memory_artifacts_e2e.rs`
- `agent_memory_loader_public.rs`

以及众多内联 `#[cfg(test)]` 模块分散在各子模块中。

---

## 十、关键设计亮点

### 1. Provenance Taint 机制
`MemoryTaint::ExternalSync` vs `Internal` 驱动潜意识门控。从外部来源（Gmail、Slack 等）摄入的 Chunk 阻止外部效果工具运行，防止从用户自己的记忆中对第三方执行误操作。

### 2. Obsidian 双向兼容
每种存储类型实现 `ObsidianRepresentable`，所有内容在用户 Vault 中显示为 `.md` 文件，用户可手动编辑，系统自动同步。降低了数据锁定风险。

### 3. 桶密封树（Bucket Seal Tree）
核心创新：不是简单的向量检索，而是**层级摘要树**。叶子追加到 L0 缓冲区，溢出时 LLM 摘要生成高层节点，实现高效的分层检索。Agent 从顶层摘要开始，按需 drill down。

### 4. 两级用户偏好
General 偏好始终注入系统提示（如用户名称、角色），Situational 偏好按轮语义检索（如当前项目上下文、近期决策）。降低 Token 消耗的同时保证上下文覆盖率。

### 5. 混合检索
70% 向量相似度 + 30% FTS5 全文搜索。向量用于语义匹配，FTS5 用于精确关键词匹配和模糊查询。

### 6. 安全隔离
工作区路径 `~/.openhuman/users/<id>/workspace` 不可写入，所有记忆数据存储在受控目录。`min_relevance_score = 0.4` 防止低质量检索干扰响应。

---

## 十一、技术债务与风险

| 问题 | 影响 |
|------|------|
| **tinycortex 过度封装** | 核心记忆逻辑在自研 crate 中，学习曲线高，调试困难 |
| **双 Cargo 世界** | memory_test 需要在两个 target/ 间协调 |
| **Ollama 依赖** | 本地 LLM 摘要/提取依赖 Ollama，用户需额外安装 |
| **云端摘要 opt-in** | `cloud_summarization_opt_in = false` 默认，大部分用户无 LLM 摘要能力 |
| **spaCy 侧车** | `spacy_enabled = true` 依赖 Python 环境，增加部署复杂度 |
| **无分布式支持** | 所有记忆在单机 SQLite 中，无法水平扩展 |
| **17 个记忆子模块** | 边界管理挑战，部分模块间存在循环依赖倾向 |

---

## 十二、总结

OpenHuman 的记忆系统是其技术深度的最佳体现。它不是简单的"文档→向量→检索"流水线，而是一个**完整的认知架构**：

```
摄入 → 分块 → 嵌入 → 树摘要 → 知识图谱 → 混合检索 → Agent 感知
```

从数据源同步（Composio/Workspace/MCP）到 Obsidian 兼容持久化，从 Provenance Taint 安全模型到两级用户偏好，从桶密封树到语义关系规则——每一层都经过精心设计。

与同类项目对比：
- 相比 **LangChain Memory**：更重、更复杂，但提供了完整的生产级记忆生命周期
- 相比 **Mem0**：存储层更丰富（SQLite + .md + 向量），但缺少 Mem0 的跨平台 SDK
- 相比 **RAGFlow**：Agent 原生集成更紧密，但缺少 RAGFlow 的 GUI 管理
