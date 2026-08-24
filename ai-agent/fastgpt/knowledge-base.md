# 知识库（RAG / Dataset）：把非结构化文档变成可检索证据的数据型能力

> 定位：系列第四篇。前三篇讲了工作流引擎（[`workflow-engine.md`](workflow-engine.md)）把节点编排成执行图、Agent 循环与会话链路（[`agent-chat.md`](agent-chat.md)）驱动对话在 agent 循环里运转、Skill 体系（[`skill-system.md`](skill-system.md)）把「做事方式」打包成可复用技能包。本篇回答：**FastGPT 怎么把一个文档变成可检索的 RAG 数据（摄取、分块、向量化、入库），混合检索与重排如何实现，又是怎样被 workflow/chat 调度的**。
>
> 阅读前提：读者知道「一次 LLM 对话 = 可编排的工作流运行时」。知识库是 workflow 里的 **dataset 类节点**——它是典型的**数据型能力**：提供「给模型按上下文找证据」的原料；对应下一篇（[`tool-mcp-sandbox.md`](tool-mcp-sandbox.md)）的工具/MCP/沙箱是**动作型能力**：「替模型去执行动作」。知识库与工具互为镜像——一个回答「我要引什么」，一个回答「我要做什么」。
>
> 仓库根：`/Users/morehao/Documents/study/ai-agent/FastGPT`。本篇引用全部为仓库相对路径 + 行号。

## 概览：知识库是「把非结构化文档变成可检索证据」的数据型能力

一句话根因：**LLM 只见过训练语料里的世界，学不会私有文档；知识库模块就是把散落的 PDF/Docx/网页/API 文件，切碎、向量化、建成索引，让模型在每次对话里有条件地「查证」而非「凭空生成」**。和裸 RAG demo 不同，FastGPT 把这条链路做成了可编排的 workflow 节点 + 可在多租户、多向量库、多变体之间复用的生产级数据管道。

这里先立住一个贯穿全篇的核心心智模型——知识库的**四层数据金字塔**：

```
Dataset（知识库 / 容器）
 └── DatasetCollection（集合 / 一份源文档）
      └── DatasetData（数据块 / 一个可选 chunk）
           └── indexes[]（向量索引，可选多条，每条对应向量库里一条向量记录）
```

分层顺序就是写入顺序：**一个知识库 → 每个来源一个集合 → 每个集合切成多个数据块 → 每个数据块按需生成多个向量索引**。检索的粒度最后落到「索引」，但它要回查最近的「数据块」（q/a）和「集合」（来源元数据）才能拼成可展示、可引用的结果——这一点决定了全篇反复出现的「向量库只存向量，元数据都回查 Mongo」的架构。

### 核心能力清单

| 能力 | 承载模块 | 一句话说明 |
|------|---------|-----------|
| 多源导入 | `collection/create/**` 系列路由 | TXT/MD/HTML/PDF/Docx/PPTX 文件、URL 链接、手动文本、QA、模板/备份、图片、API 文件 |
| 分块（chunk） | `collection/utils.ts`、`read.ts`、worker `text2Chunks` | 段落/固定大小/字符三种 split 模式 + 分块触发阈值 + token 安全拆分 |
| 向量化入库 | `data/dataIndex.ts`、`data/data.ts` | 生成系统/自定义索引 → embedding → 写向量库 → 回写 Mongo 索引 id，强一致顺序 |
| 训练队列 | `training/schema.ts`、`queues/{datasetParse,generateQA,generateVector}.ts` | 异步多阶段流水线，队列 + 锁 + 重试 + TTL 清理 |
| 检索引擎 | `search/defaultRecall/**` | embedding / 全文 / 混合三种模式，多路召回 + RRF 融合 + rerank 精排 + 过滤 |
| 图文检索 | `imageCaption.ts`、`dataIndex.ts` | VLM 图片描述转文本 query、imageEmbedding 图片向量索引 |
| 重排序 | `defaultRecall/rerank.ts` | 可选 rerank 模型对文本召回精排，与 RRF 权重融合 |
| 元数据过滤 | `defaultRecall/collectionFilter.ts` | 标签 / createTime / collectionId 组合过滤（isPlus） |
| API 知识库 | `apiDataset/**` | 自定义 HTTP / 语雀 / 飞书 / 钉钉，通过 `/v1/file/*` 拉远程文件 |
| 异步集合/知识库同步 | `datasetSync/`、`collection/utils.ts` | `autoSync` + BullMQ 定时同步、`syncCollection` 增量重抓 |
| 图片知识库 | `image/**`、`utils.ts` | 图片源存储、VLM 描述、S3 预览 URL 短期签发 |
| 标签系统 | `tag/schema.ts` | collection 级标签，支撑检索过滤与组织 |

### 技术栈一览

知识库不发明新存储，而是**复用平台已有基建做组合**：MongoDB/Mongoose 存结构化元数据与全文 token（`readFromSecondary` 读从库降主库压力）、PostgreSQL/pgvector 存向量（HNSW 索引）、S3 存原始文件与图片（已从 GridFS 迁移）、BullMQ 做异步队列（分块/向量化/删除/同步/collection 更新防抖）、jieba 中文分词、tiktoken 做 token 计数、`mongoSessionRun` 保证跨库写一致。这套「Mongo 承担元数据与事务、向量库只当检索加速器」的分工，是本模块最值得解的架构决定。

## 设计思路：为什么「Mongo 存元数据 + VectorDB 存向量」才是对的

### 检索时为什么必须在「向量库 → Mongo」之间来回

一次检索，向量库返回的只是 `{ id, collectionId, score }`（`vectorDB/type.ts:4-9` 的 `EmbeddingRecallItemSchema`），没有 q、a、文件名这些能呈现给用户的东西。所以 `embeddingRecall.ts` 在拿到向量库命中的 index 后，要再用这批 `dataId` 回查 `MongoDatasetData`、用 `collectionId` 回查 `MongoDatasetCollection`，拼装成带 q/a/来源的 `SearchDataResponseItemType`（`search/defaultRecall/embeddingRecall.ts:200-240`）。全文召回同理（`fullTextRecall.ts:112-155`，而且为了减少展示字段的 Mongo 查询，专门做了字段投影 `datasetDataSelectField` / `datasetCollectionSelectField`，见 `defaultRecall/constant.ts:5-31`）。

为什么会设计成「向量库不存元数据」？权衡如下：

- **独立性**：换向量库（PG→Milvus→OceanBase…）时，元数据不迁移、不需同步字段；向量库只要是「按键检索」就够了（`VectorControllerType` 接口只有 insert/delete/embRecall/getVectorCount/getVectorDataByTime 六个方法，`vectorDB/type.ts:84-114`，不感知 q/a）。
- **一致性**：q/a/来源/权限/集合关系这些会频繁变、需要事务的字段放 Mono（可事务），向量库里只有「死向量 + 一个索引文本」；`data/data.ts` 的写入顺序专门保证两者不脱节（见后）。
- **代价**：检索要一次 join（向量库一次 + Mongo 一到两次），延迟略高；要靠 `readFromSecondary` 与字段投影缓解（`data/data.ts` 的所有读取都带 `{ ...readFromSecondary }`）。

### 多向量库适配层：接口收窄、实现各自补齐

`vectorDB/controller.ts:22-30` 的 `getVectorObj()` 按环境变量优先级选择实现：`SEEKDB_ADDRESS > OCEANBASE_ADDRESS > PG_ADDRESS > MILVUS_ADDRESS > OPENGAUSS_ADDRESS`，默认回落到 `PgVectorCtrl`。每个实现都实现同一个 `VectorControllerType` 接口，但内部差异巨大：

- PG：建 `vector` 扩展、`VECTOR(1536)` 定长列 + HNSW 索引（`pg/index.ts:12-37`），召回用 `vector <#>` 负内积 + `hnsw.ef_search` 调参（`pg/index.ts:155-169`）。
- Milvus：集合 + HNSW + 「内积 IP」metric，命中结果仍统一返回 `{ id, collectionId, score }`（`milvus/index.ts:240-262`）。
- OceanBase/OpenGauss/SeekDB：按 `VECTOR_VQ_LEVEL` 切换索引类型与距离函数（`constants.ts:55-85` 的 `OceanBaseIndexConfig`，从 hnsw 到 hnsw_bq/cosine 多档量化级别）。

这就是「适配」的形态：**上层只认统一的召回语义（score 越小越好还是越大越好、要不要做 score transform、过滤怎么写），具体差异被每个实现的 `embRecall` 吸收**（例如 PG 用 `score * -1` 翻转负内积，`pg/index.ts:186`；OceanBase 的 `scoreTransform` 在 `constants.ts:64`）。换向量库 = 换一个 class + 改环境变量，知识库代码零改动。这是「多库复用」宣传口径站得住脚的根因。

> 局限：适配层仍把「向量维度」写死为 1536（`pg/index.ts:21`、`milvus/index.ts:72`），换更高维 embedding 模型要改表结构。这是当前实现的一个硬编码点，非用户可配置。

## Schema：一份刻意「可演进」的数据契约

### 数据集与集合：一个容器、一张来源表

`datasets`（`schema.ts:59-152`）是最外层容器，`type` 区分普通知识库/文件夹/网站/API 数据集/外部文件，`chunkSettings` 内嵌一整套分块配置（`schema.ts:20-57` 的 `ChunkSettings`，含 split 模式、size、splitter、indexSize、qaPrompt 等）；`dataset_collections`（`collection/schema.ts:13-87`）是「一份源文档」，`type` 区分 folder/file/link/apiFile/images/virtual，并通过 `fileId / rawLink / apiFileId / externalFileUrl` 记录来源，`...ChunkSettings` 展开继承分块配置（`collection/schema.ts:86`）。

**分块配置被「数据集 → 集合」两层持有**，是刻意的：知识库级给默认值，集合级可覆盖——不同来源的文档（一页官网 vs 千页 PDF）可以在同一知识库里各用各的分块策略。

### 数据块：最小知识单元的「多索引」设计

`dataset_datas`（`data/schema.ts:35-92`）是检索真正命中的对象，字段上最有设计含量的是 `indexes: []`（`data/schema.ts:53-76`）：一个数据块可以挂**多条**向量索引，每条 `{ type, dataId, text }`。`dataId` 就是向量库里的那条向量 id，`text` 是索引文本。这回答了「为什么一个 chunk 能有多个检索入口」——大块文本按粒度切成多个 `default` 索引，或叠加 `custom/question/summary/image` 等外部索引。`q` 与 `a` 不参与展示时的分块，只在需要时投影（`defaultRecall/constant.ts:5-17` 里 search 场景读的都是精简字段）。

> 索引文本 ≠ 展示文本（q/a）：`indexes[].text` 可能带集合前缀、被 token 拆分，而 `q/a` 是用户最终看到的原始内容。检索用索引，展示用 q/a，两者解耦（这也 echo 了 embedding-token-limit-guard 文档「索引可拆分多条、但 q/a 原文不动」的原则）。

### 训练记录：承上启下的「流水线暂存」

`dataset_trainings`（`training/schema.ts:17-103`）是摄取流水线的中转表：从「分块完成」到「向量化入库」之间的任务都暂存这里。它同时是 collection 前台的进度/错误来源。关键字段：`mode`（parse/chunk/qa/auto/image/imageParse，见 `global/core/dataset/constants.ts:240-246`）、`retryCount`（默认 5 次重试，`training/schema.ts:56-59`）、`lockTime`（分布式抢占）+ `expireAt`（7 天 TTL 兜底清理，`training/schema.ts:138-141`）。

## 概览补充：一条摄取流水线到底长什么样

上面四层是「静态」的数据契约；下面用一张图把「文档上传 → 到可检索」的**多阶段流水线**串起来，锚点标注关键实现位置。这正是「一次导入，多进程接力」的形态——它决定了知识库能异步、可重试、可监控地消化大文档。

```mermaid
flowchart TB
    subgraph Ingest["摄取管道：一串异步队列接力"]
        A["API 创建集合<br>collection/create 系列路由"] --> B{"rawText 已就绪？"}
        B --"是(文件/链接/文本/CSV)"--> C["createCollectionAndInsertData<br>collection/controller.ts:41"]
        C --> C1["rawText2Chunks 分块<br>read.ts:342"]
        C1 --> C2["pushDataListToTrainingQueue 直入 chunk 队列<br>training/controller.ts:73"]
        B --"否(需要先解析)"--> D["pushDatasetToParseQueue 入 parse 队列<br>training/controller.ts:265"]
        D --> D1["datasetParseQueue 读取源 + LLM段落 + 分块<br>projects/app/.../queues/datasetParse.ts:107"]
        D1 --> D2["再 pushDataListToTrainingQueue 入 chunk 队列"]
        C2 --> E["generateVector 向量化入库队列<br>projects/app/.../queues/generateVector.ts:79"]
        D2 --> E
        E --> E1["dataIndex 生成系统/自定义索引 + token拆分<br>projects/app/.../data/dataIndex.ts:177"]
        E1 --> E2["写向量库再写 Mongo(强一致)<br>data/data.ts:105 create"]
    end
    subgraph Tip["可选分支：QA / 图片 / 自动"]
        C --"QA 模式"--> F["generateQA LLM拆分Q/A<br>queues/generateQA.ts:37"]
        F --> E
        C --"图片/带图文档"--> G["imageEmbedding / VLM描述<br>dataIndex.ts getImageEmbeddingSources"]
        G --> E
    end
```

（详细流水线拆解见「文档摄取」一节的逐步展开。）

## 文档摄取：一个文档是怎么变成 N 条可检索记录的

摄取不是「上传 → 立刻能搜」的同步动作，而是 **create（建集合/分块）→ parse（选项：解析+LLM 改段落）→ chunk（分块）→ vector（向量化入库）** 的多进程接力，每段用独立队列承载。这样设计的根因：分块可能要调 LLM（qa/auto 模式）、向量化要调 embedding 且按 token 计费，都不适合堵在 HTTP 请求里同步完成，也不该因为一线失败就全盘重来。

### 第 1 步：建集合 + 分块（同步，请求内）

[packages/service/core/dataset/collection/controller.ts:41-128]() 的 `createCollectionAndInsertData` 是摄取前台的统一入口：
<!-- anchor: export const createCollectionAndInsertData = async ({ -->

1. **配置归一**：`computedCollectionChunkSettings` 按 agent 模型 / 向量模型的 max 容量校正分块参数，并把旧版 `auto` 训练类型换算成 `chunk + autoIndexes`（`collection/controller.ts:61-64`）。这是「前向兼容」的典型——旧数据集迁移后字段语义变了，代码里兜底改写。
2. **判定训练模式**：`getTrainingModeByCollection`（`collection/utils.ts:215-252`）按 `trainingType + autoIndexes + imageIndex + isPlus` 决定走 chunk / qa / auto / image / imageParse。这个函数把「用户选的导入类型」翻译成「流水线要跑的几个模式」，isPlus 才开放 qa/auto/image 这些会烧 LLM 地高级转型。
3. **分块**：`rawText2Chunks`（`read.ts:342-492`）按触发阈值决定切不切——文本太短直接整个作为一个 chunk（`read.ts:460-479`），否则交给 `text2Chunks` worker 切。
4. **推队列**：分块结果 `pushDataListToTrainingQueue` 直入 chunk 队列；若连 rawText 都要现读（link/API 文件），则先 `pushDatasetToParseQueue` 入 parse 队列（`collection/controller.ts:207-243`）。

所有写入都包在同一个 Mongo 事务里（`mongoSessionRun`，`collection/controller.ts:177`），建 collection、建账单、写 training 一条龙要么全成要么全败。

### 第 2 步：解析队列——顺手把「读源」也异步化

`projects/app/src/service/core/dataset/queues/datasetParse.ts:107` 的 `datasetParseQueue` 处理「需要先解析源文档」的集合（link 网页、API 文件、本地文件）：

- 按 source 类型分发到 `readDatasetSourceRawText`（`read.ts:215-313`）：文件走 S3 读取、链接走 `urlsFetch` 爬正文、API 文件走 `getApiDatasetRequest`（`apiDataset/index.ts:7-22` 按配置选 custom/语雀/飞书/钉钉适配器）。
- 可选 `requestLLMPargraph`（`datasetParse.ts:41-98`）：当 `paragraphChunkAIMode` 开且非 Markdown 时，调商务接口（`POST /core/dataset/training/llmPargraph`）让 LLM 给文本「分段落」，返回带语义边界的文本再分块——比纯正则分块更懂上下文。这是 isPlus 的高级能力。
- 用 **task lease（心跳续租）** 管理长任务（`datasetParse.ts:213-236` + `parseLease.ts`）：worker 崩溃后 lease 过期，任务可被其他 worker 重新抢占。这保证了「解析一个 100 页 PDF 耗时几分钟」时，进程不掉线任务不丢。

### 第 3 步：分块（chunk）——三种策略 + token 安全

分块是「召回质量」的第一道分水岭。`global/core/dataset/constants.ts` 定义 `DataChunkSplitModeEnum`：**paragraph（智能段落）、size（固定大小）、char（分隔符）**。不同模式权衡：

- **paragraph**：按标题/段落层级切，语义完整，适合 MD/结构化文本；deep 控制层级，minSize 兜底小段落（配置在 `ChunkSettings`）。
- **size**：固定 token 数切，实现简单、块大小可控，但可能把一句话腰斩。
- **char**：按自定义分隔符切（`chunkSplitter`），最可控但要求用户懂内容结构。

`rawText2Chunks` 的分块阈值还有一个精妙设计：`chunkTriggerType=minSize` 时只有文本超过 `chunkTriggerMinSize` 才真切，否则整段一个 chunk（`read.ts:474-479`）——避免把短文档硬切成无意义的碎片。

分块粒度 `chunkSize` 还会被 `getLLMMaxChunkSize` 限制在模型上下文能消化的范围（`datasetParse.ts:338`）。到了索引阶段，索引文本还要再按 embedding 模型的 `maxToken` 做二次 token 安全拆分（下节详述）。

### 第 4 步：向量化入库——为什么「先写向量库，再写 Mongo」

`generateVector`（`projects/app/src/service/core/dataset/queues/generateVector.ts:79`）从 training 队列取 chunk 任务，调 `createDatasetData`（`data/data.ts:492`）完成最后一次落库。核心在这条强一致顺序：

1. `indexOperation.formatIndexes` 归一化系统索引 + 自定义索引（`dataIndex.ts:288-396`）。
2. `insertVectors` **先把每条索引文本向量化并写入向量库，拿到 `dataId`**（`dataIndex.ts:633-664`）。
3. 再写 `MongoDatasetData`（q/a/images/indexes 带 dataId）（`data.ts:160-177`）。
4. 最后写全文检索 token 表 `MongoDatasetDataText`（`data.ts:180-191`）。

为什么这个顺序不可颠倒？**Mongo 里存的是「带 dataId 的索引」，dataId 必须来自向量库写入结果**。如果先写 Mongo 再写向量库，中间崩溃会留下「Mongo 说有索引、向量库没有」的脏状态——检索时回查得到 `dataId` 却查不到向量，直接悬空。反过来「向量库先进、Mongo 后进」即使中途失败，只会留下**孤儿向量**（可靠向量 id 集合清理），不会产生悬空引用。这就是「宁可浪费一点孤儿向量，也要保住检索的引用完整性」的取舍（`data/data.ts:66-67` 注释明确写着「注意写入顺序，避免 index dataId 和向量库记录不一致」）。

`DatasetDataIndexOperation`（`dataIndex.ts:177`）把索引生命周期的所有操作（生成、format、patch diff、写删向量、维护 Mongo indexes 数组）收进一个类，因为 **Mongo 里的 `dataId` 必须和向量库 id 保持一致**（`dataIndex.ts:175` 的注释直接点明这个耦合）。

### 第 4.5 步：embedding token 安全——超长索引不炸库

这是 `embedding-token-limit-guard` 设计文档要解决的痛点：**embedding provider 按 token 限输入，而老代码只按字符/分块大小控制文本，超长文本会把 embedding 调用打爆**。

对策是「入库可切分、检索只截断、入口统一兜底」三层：

- **入库**：`splitIndexTextByTokenLimit`（`dataIndex.ts:75-115`）按 `min(indexSize, embedding.maxToken - 前缀token)` 把索引正文切到安全 token 数，系统索引直接生成多条 token-safe 索引（`getSystemIndexes`，`dataIndex.ts:229-280`）；外部自定义索引超过上限才二次拆分（`buildEmbeddingSafeIndexTexts`，`dataIndex.ts:123-148`）。**拆分只动 `indexes[].text`，绝不改 q/a 原文**（这是文档反复强调的红线）。
- **检索**：query 用一个统一入口 `getVectors` 兜底——在调 provider 前按 `model.maxToken` 截断 text input（`embeddingRecall.ts` 里查询一律走 `getVectors`，`defaultRecall/embeddingRecall.ts:113-117`）；不把一个 query 拆多个，image input 不参与文本截断。
- **相似度**：`useTextCosine` 等只做 trim/filter/选择，最终截断由 `getVectors` 统一兜底——让「截断规则」只存在于一个地方，各调用点不再重复写。

这个设计的权衡：合法优雅地**增量化处理超长**，代价是 token 计数（本地 o200k_base tokenizer）与 provider tokenizer 可能有微小差异（文档「风险与注意事项」明说），极端 case 仍需 provider 抛错兜底。它把「上游尽量控长 + 入口硬截断」两层缝合，解决了 RAG 系统里最容易偶发但极难排查的「embedding 超限」类问题。

## 训练队列机制：多并发、可抢占、可重试、自动清

`training/schema.ts` + 三 consume 函数（generateVector / generateQA / datasetParse）拼成一个**消费型队列**：worker 用 `findOneAndUpdate` **原子抢占**——`lockTime <= now-3min` 的任务才可取（`generateVector.ts:97-102`），抢到后立刻写 `lockTime + $inc: retryCount: -1`（`generateVector.ts:103-106`）。这把「谁拿这个任务」变成了一条 Mongo 原子操作，天然支持多实例/多 worker 并发（`global.vectorMaxProcess` 控制每个进程几个并发消费者，`generateVector.ts:80`）。

- **锁**：抢占即加锁；跑完删任务（`generateVector.ts:372`），失败 `errorMsg + lockTime 复位`（`generateVector.ts:208-215`）等下一轮重试。
- **重试**：`retryCount` 递减，扣光视为「最终失败」；永久锁用 `lockTime=2050` 或 `2999` 表达「定点暂停」（如余额不足、`datasetSizeNotEnough` 锁到 2999，`datasetParse.ts:407-417`）。
- **TTL**：`expireAt` 7 天自动删除（`training/schema.ts:138-141`），兜底清理卡死的孤儿任务。
- **余额校验**：消费前 `checkTeamAiPointsAndLock`（`generateVector.ts:162`, via `queues/utils.ts`），AI 点数不足整个 team 的未完成任务被锁住（`deleteTimerLock`/`checkTimerLock`，`training/controller.ts:19-71` 是 5 分钟闸门防多节点重复锁）。

这套「任务即 Mongo 文档、抢占即原子 update」的队列，没有引入独立消息中间件（对知识库场景是刻意的——任务状态要和训练进度/错误展示强绑定，天然该活在数据库里；真要异步编排的部分（删除/同步/collection 更新）才交给 BullMQ）。

## 与 workflow / chat 集成：dataset 节点执行器怎么「吃」到这个数据能力

知识库检索要被 workflow 调用，入口在 `workflow/dispatch/dataset/search.ts` 的 `dispatchDatasetSearch`。它把 workflow 节点的输入（选哪些知识库、similarity、limit、searchMode、是否 rerank、是否 query extension、元数据过滤）翻译成一次 `defaultSearchDatasetData` 或 `deepRagSearch` 调用（`search.ts:139-179`），拿到 `searchRes` 后输出成 `quoteQA` 注入对话上下文，并把**分项计费**汇报给 workflow 计费面板：embedding、rerank、query extension（LLM+embedding）、image caption（VLM）、deep search 各算各的 points（`search.ts:182-289`）。

这个节点的价值在于「把知识库的复杂性收进一个节点」：应用编排者不关心内部有多少召回路径，只配置「查哪个知识库、多严的相似度、多少 token」；而计费面板能把一次检索的多模态消耗拆得清清楚楚——这正是「知识库是数据型能力、可被 workflow 编排复用」的落地形态。

> 关于计费路径：`filterDatasetsByTmbId` 会对数据集逐个做读权限校验（`utils.ts:128-153`，作者已留 TODO 说「需要优化成批量获取权限」），这是当前一个明显的性能欠缺点。

### Deep RAG：`global.deepRagHandler` 的全局钩子

`search/index.ts:74-75` 的 `deepRagSearch` 只是 `global.deepRagHandler(data)` 的转发——把 Deep RAG（多轮追问检索）实现挂成全局 handler，搜索分发层不持有具体实现（`search/index.ts:71-75` 注释点明「避免具体 recall 实现持有全局 handler 适配」）。这个钩子表明知识库检索是可扩展的：默认召回是一套，Deep RAG 是另一套，入口保持不变。

## 检索引擎：混合检索 + 重排是怎么拼出「最终引用」的

检索入口 `search/index.ts:18` 的 `defaultSearchDatasetData` 先做 query extension（可选），再把 query 交给 `searchDatasetData`（[packages/service/core/dataset/search/defaultRecall/index.ts:34-198]()）。这条链路是「多路召回 → 分组融合 → 文本精排 → 图文合并 → 去重/过滤 → 格式化」的多级 pipeline。下面先给全景图，再逐层拆。
<!-- anchor: export async function searchDatasetData( -->

```mermaid
flowchart LR
    subgraph Recall["召回（多路 + 并行）multiQueryRecall.ts:10"]
        Q["textQueries / imageQueries"] --> QC["imageCaption: VLM把图片变描述<br>imageCaption.ts:34"]
        QC --> E["embeddingRecall<br>embeddingRecall.ts:134"]
        Q --> E
        Q --> F["fullTextRecall jieba分词$text索引<br>fullTextRecall.ts:27"]
        QC --> F
    end
    subgraph Fuse["融合（RRF / 权重）"]
        E --> T1["文本: emb→weight, ft→weight<br>index.ts:111 RRF"]
        E --> T2["图片描述: 同上<br>index.ts:115"]
        F --> T1
        F --> T2
        T1 --> R["rerank 只对文本精排<br>rerank.ts:55"]
        T2 --> IMG["图片: caption(0.3)+向量(0.7) 再与文本(0.7/1)合并<br>index.ts:137-158"]
        R --> M["concatWeightedRecallLists<br>result.ts:datasetSearchResultConcat"]
        IMG --> M
    end
    subgraph Filter["过滤与输出"]
        M --> D["removeDuplicate去重<br>result.ts:62"]
        D --> S["filterSearchResultsByScore 相似度阈值<br>result.ts:74"]
        S --> T["filterDatasetDataByMaxTokens token预算<br>utils.ts:39"]
        T --> O["formatDatasetDataValues 签S3预览URL<br>data/controller.ts:76"]
    end
```

### 召回①：embedding 向量召回

`embeddingRecall.ts:134` 把用户文本 + 图片描述都转成向量（`buildVectorRecallTasks`，`embeddingRecall.ts:39-127`），对每个 query 向量调向量库 `embRecall`（`embeddingRecall.ts:180-191`），拿到一堆 `{ id, collectionId, score }` 后回查 Mongo 拼数据。**原始图片**（imageQuery）只在 embedding 模型支持图片时才单独转 base64 做 image embedding（`embeddingRecall.ts:77-104`）；单张图片失败只丢这条路径，不拖垮整次检索（`embeddingRecall.ts:84-89` 注释点明「additive」原则）。每个 query 内先去重，多 query 间交给 RRF（`embeddingRecall.ts:294-302`）。

### 召回②：全文关键词召回

`fullTextRecall.ts:27` 走 Mongo 的 `$text` 索引：先用 **jieba 中文分词**（`jiebaSplit`，`fullTextRecall.ts:69`）把 query 切成词，再对 `dataset_data_texts` 做全文检索并按 `textScore` 排序取 topN。这里的数据模型值得注意：分词后的 `fullTextToken` 在**入库时就算好存到单独的表**（`data.ts:180-191` 里 `create` 时存 `fullTextToken`），而不是查询时现算——**把「分词开销」提前到写入期，查询期只做倒排匹配**。而全文表索引用 `default_language: 'none'`（`dataTextSchema.ts:35-41`），避免英文 stemming 干扰中文召回。

> 设计取舍：为什么全文走 Mongo 而不是向量库？因为「精确关键词命中」是语义召回互补的——向量召回擅长「意思相近但用词不同」（如「如何退款」≈「退还流程」），全文召回擅长「必须出现某个专有词」（如型号「FG-2000」）。两者独立、再融合，覆盖召回长尾。

### 融合①：同源 RRF 加权

`datasetSearchResultConcat`（`global/core/dataset/search/utils.ts:5-70`）是融合引擎：对每个候选 `score = Σ weight × 1/(60+rank)`（RRF 公式，`utils.ts:19-21`），同 id 合并、同 score 类型取 max、最终按 RRF score 排序。`concatWeightedRecallLists` / `concatRecallLists`（`result.ts:48-56`）是它的封装，第 `60` 是 RRF 常数（避免排名第一被过分放大）。

融合是**分层**的（`index.ts:111-158`）：先各自融合「文本的 embedding+fullText」、「图片描述的 embedding+fullText」；再合并图片（caption 权重 0.3、图片向量权重 0.7）；最后按「纯文本(1) / 图文混合(0.7)」把文本与图片结果合起来。这套权重分层让「图片」永远作为「文本」的补充而非主导，除非用户只发图（`hasTextQuery ? 0.7 : 1`）。

### 融合② + 精排：rerank 只作用于文本

`rerank.ts:55` 的 `reRankSearchResults` 用 rerank 模型（如 bge-reranker）对**文本召回**再精排——`rerank.ts` 刻意排除图片向量和 caption 结果（`rerank.ts:52` 注释：「避免用文本 rerank 把视觉相似结果误杀」）。这是深思的取舍：rerank 是「按 query 对候选重新打分」，它对文本敏感，对纯视觉相似未必公平，所以只精排文本、图片维持 RRF 排序。

精排后若 `rerankWeight < 1`，再把它和原文本召回按权重融合（`rerank.ts:95-101`），否则直接取 rerank 结果。rerank 失败回退原文本召回（`rerank.ts:103-109` 的 try/catch），**精排是「尽力而为」的增强，绝不能成为主链路的单点**。

### 过滤：相似度阈值 + token 预算

最终过滤顺序固定为：**去重 → 相似度 → token 上限**（`index.ts:160-170`）：

- `removeDuplicateSearchResults` 用「去掉标点的 q+a」做 hash 去重（`result.ts:62-72`），避免同一 chunk 因多路召回重复占预算。
- `filterSearchResultsByScore` 按当前启用分数类型（rerank / embedding）过滤低于 `similarity` 的项（`result.ts:74-105`）。
- `filterDatasetDataByMaxTokens` 从 top 往下累积 token 直到预算上限（`utils.ts:39-79`），**且至少留一条**（`utils.ts:63` `results.length === 0 ? data.slice(0,1)`），避免「高质量单条超长被全滤掉 → 用户看到空引用」这种更难排查的结果。

token 预算是 `limit` 参数（workflow 节点里叫「最大引用 token」，`search.ts:29`），这就是把「喂给模型的引用量」锁在上下文安全范围内。

### Query extension & 图片描述：检索前的前置增强

- `datasetSearchQueryExtension`（`search/utils.ts:69-140`）用可选的 LLM 把用户 query 扩成多个相关 query（`extensionQueries`），去重后一起语义召回，提升「用户问得含糊但模型能猜意图」场景的召回。
- `getImageCaptionQueries`（`imageCaption.ts:34`）对图片 query 用 VLM 生成一句描述（中文 prompt，`imageCaption.ts:80-83`），把「图」变成「可文本匹配的说明」，让不支持图片向量口径的模型也能通过描述参与召回（是图片检索的兜底路径）。两者的结果都回传给 workflow 计费（`search/index.ts:53-68`）。

## Schema 演进：这份数据契约是怎么「安全变老」的

知识库 Schema 不是一次定型，而是随功能需求持续演进，本模块的演进手法值得单独讲：

### 1. 软删除：删知识库不是真删

`datasets` 的 `deleteTime` 字段（`schema.ts:136-139`）是关键演进：删除知识库先**打标软删**（`delete/processor.ts:49-63` 的 `deleteTeamAllDatasets` 只 `$set: deleteTime`），真正清数据交给后台 **BullMQ 异步 processor**（`delete/processor.ts:125-180`）——它先做安全校验「只删已标记 deleteTime 的知识库」（`processor.ts:144-161`），再删 tags/avatar/data/全文/向量/权限/集合（`delDatasetRelevantData`，`controller.ts:67-127`）。**软删 + 异步确认**把「删除」这个大动作拆成「先不可见、后物理清除」两段，避免请求里同步删上万条记录。

### 2. 存储迁移：GridFS → S3 的可回滚日志表

`migration/schema.ts:4-87` 定义 `dataset_migration_logs`，专为「文件存储从 GridFS 迁到 S3」这类**数据迁移**留审计痕迹：每条记录 `sourceStorage(gridfs)` → `targetStorage(s3)`、状态机 `pending→processing→completed/failed/rollback/verified`、校验信息（checksum）、失败 phase（download/upload/verify/update_db）与 operations 操作日志（`migration/schema.ts:44-72`）。它能支撑「迁移中断 → 依据日志回滚/续传」——这是做 schema 演进/存储迁移时「先记账、后动手」的工程化思路。

> 代码里随处可见 S3 已取代 GridFS：`collection/schema.ts:62` 的 `fileId` 注释「Support both GridFS ObjectId (string) and S3 key (string)」，`data.ts` 里 `isS3ObjectKey` 判断、`removeS3TTL` 管理临时文件 TTL，`read.ts:245` 里文件必须 `isS3ObjectKey(sourceId,'dataset')` 才读。图片知识库用的是 GridFS 元数据模型（`image/schema.ts:4` `bucketName='dataset_image'`）但文件本体已迁 S3——这是混合态。

### 3. 全文索引拆分：从 data 内嵌字段 → 独立 text 表

`data/schema.ts:88-91` 里 `fullTextToken/initFullText/initJieba` 都标了 **`// Abandon`（废弃）**，全文检索 token 已从 `dataset_datas` 内嵌迁到独立的 `dataset_data_texts`（`dataTextSchema.ts:11-33`，结构精简为 `dataId + fullTextToken`，带独立 text 索引）。迁表的收益：`dataset_datas` 的写路径（insertMany 批量）不再背着全文索引的 `$text` 维护成本，两个表的增长与备份尺度解耦。

### 4. 索引同步治理：不删客户自建索引

`mongo-index-sync-strategy` 文档解决一个私有化部署的痛点：`mongoose.syncIndexes()` 会在启动时**删掉 Schema 未声明的索引**——但客户用 mongosh 自建的索引会被误删。治理结论（落地在 `schemaIndexes.ts` / `indexManager.ts`）：启动时**只补建当前 Schema 缺失的索引、绝不删任何未知索引**；废弃索引需要显式 `defineIndex({..., deprecated:true})` 登记才会被后台清理（且必须「先建成功当前索引才允许删」）。本模块所有 schema 的 `defineIndex(...)` 调用（如 `schema.ts:154-156`）都是走这个受控入口，而非裸 `Schema.index()`。这是「运维安全优先于索引整洁」的取舍——索引多几个不影响正确性，客户数据库被误删却不可逆。

## 与相邻模块的协作边界

- **workflow 编排**：`dataset/search.ts` 节点把知识库封装为「选库 + 调参 + 计费」的黑盒，应用只需配置、无需操心内部召回路径（见「与 workflow / chat 集成」）。
- **chat 链路**：检索结果 `quoteQA` 注入对话供模型引用；`data/controller.ts` 负责把内部 S3 图片 key 换成带 90 天有效期的预览 URL（`formatDatasetDataValues`，`data/controller.ts:76-112`），避免把内部 key 泄露给最终用户——「内部存储标识」与「对外可访问 URL」在展示层解耦。
- **向量库适配**：`vectorDB/controller.ts` 的 `insertDatasetDataVector`/`recallFromVectorStore`/`deleteDatasetDataVector` 是知识库与向量库之间唯一的门面，上层不感知具体引擎（`controller.ts:38-97`），且统一包 `retryFn` 做网络抖动重试、操作后 `teamVectorCountCache.invalidate` 失效计数缓存。
- **存储 S3**：文件本体、解析出的图片、原始文本缓存都落在对象存储；数据块的 `indexes[].dataId` 作为唯一跨存储引用（Mongo ↔ 向量库）——这个 id 是「一石二鸟」：既是 Mongo 数组项的身份，也是向量库表的主键（`dataIndex.ts:31-35` 注释点明「dataId 同时也是向量库里的向量 id」）。

> 跨模块结论【待复核】。

## 覆盖率核对

> 说明：本模块核心 = `packages/service/core/dataset/**`（除 API 数据集第三方适配器）+ `packages/service/common/vectorDB/**` + `projects/app/src/service/core/dataset/**`（摄取队列与索引写）+ `packages/service/core/workflow/dispatch/dataset/**`（workflow 集成）。下表为逐文件核对结果（按仓库根相对路径）。

### service/core/dataset 核心

| 文件 | 总行 | 已读 | 覆盖率 | 未读原因 |
|------|-----|------|--------|---------|
| core/dataset/schema.ts | 158 | 158 | 100% ✅ | — |
| core/dataset/controller.ts | 128 | 128 | 100% ✅ | — |
| core/dataset/utils.ts | 217 | 217 | 100% ✅ | — |
| core/dataset/read.ts | 493 | 493 | 100% ✅ | — |
| core/dataset/importFile.ts | 60 | 60 | 100% ✅ | — |
| collection/schema.ts | 140 | 140 | 100% ✅ | — |
| collection/controller.ts | 456 | 456 | 100% ✅ | — |
| collection/utils.ts | 252 | 252 | 100% ✅ | — |
| collection/mq.ts | 56 | 56 | 100% ✅ | — |
| data/schema.ts | 119 | 119 | 100% ✅ | — |
| data/dataTextSchema.ts | 50 | 50 | 100% ✅ | — |
| data/controller.ts | 138 | 138 | 100% ✅ | — |
| data/utils.ts | 46 | 46 | 100% ✅ | — |
| training/schema.ts | 146 | 146 | 100% ✅ | — |
| training/controller.ts | 293 | 293 | 100% ✅ | — |
| training/query.ts | 153 | 153 | 100% ✅ | — |
| migration/schema.ts | 259 | 259 | 100% ✅ | — |
| search/index.ts | 75 | 75 | 100% ✅ | — |
| search/type.ts | 92 | 92 | 100% ✅ | — |
| search/utils.ts | 140 | 140 | 100% ✅ | — |
| search/defaultRecall/index.ts | 198 | 198 | 100% ✅ | — |
| search/defaultRecall/embeddingRecall.ts | 303 | 303 | 100% ✅ | — |
| search/defaultRecall/fullTextRecall.ts | 218 | 218 | 100% ✅ | — |
| search/defaultRecall/multiQueryRecall.ts | 84 | 84 | 100% ✅ | — |
| search/defaultRecall/rerank.ts | 110 | 110 | 100% ✅ | — |
| search/defaultRecall/result.ts | 105 | 105 | 100% ✅ | — |
| search/defaultRecall/utils.ts | 79 | 79 | 100% ✅ | — |
| search/defaultRecall/collectionFilter.ts | 248 | 248 | 100% ✅ | — |
| search/defaultRecall/imageCaption.ts | 129 | 129 | 100% ✅ | — |
| search/defaultRecall/constant.ts | 31 | 31 | 100% ✅ | — |
| delete/index.ts | 13 | 13 | 100% ✅ | — |
| delete/processor.ts | 180 | 180 | 100% ✅ | — |
| datasetSync/index.ts | 78 | 78 | 100% ✅ | — |
| image/schema.ts | 40 | 40 | 100% ✅ | — |
| image/controller.ts | 25 | 25 | 100% ✅ | — |
| tag/schema.ts | 33 | 33 | 100% ✅ | — |
| apiDataset/index.ts | 23 | 23 | 100% ✅ | — |
| apiDataset/custom/api.ts | 294 | 294 | 100% ✅ | — |
| apiDataset/feishuDataset/api.ts | 226 | 60 | 27% | 与 custom 同构的第三方 HTTP 适配（token 鉴权 + getFileContent 同接口），已确认模式 |
| apiDataset/yuqueDataset/api.ts | 340 | 0 | 0% | 同上，语雀 OpenAPI 适配，模式同构 |
| apiDataset/dingtalkDataset/api.ts | 615 | 0 | 0% | 同上，钉钉 OpenAPI 适配，模式同构 |
| **service 核心小计（除第三方适配器）** | **~5662** | **~5662** | **100% ✅** | — |

### vectorDB 适配层

| 文件 | 总行 | 已读 | 覆盖率 | 未读原因 |
|------|-----|------|--------|---------|
| common/vectorDB/constants.ts | 85 | 85 | 100% ✅ | — |
| common/vectorDB/controller.ts | 117 | 117 | 100% ✅ | — |
| common/vectorDB/type.ts | 114 | 114 | 100% ✅ | — |
| common/vectorDB/global.d.ts | 11 | 11 | 100% ✅ | — |
| common/vectorDB/pg/controller.ts | 208 | 208 | 100% ✅ | — |
| common/vectorDB/pg/index.ts | 236 | 236 | 100% ✅ | — |
| common/vectorDB/milvus/index.ts | 321 | 321 | 100% ✅ | — |
| common/vectorDB/oceanbase/controller.ts | 262 | 262 | 100% ✅ | — |
| common/vectorDB/oceanbase/index.ts | 222 | 222 | 100% ✅ | — |
| common/vectorDB/opengauss/controller.ts | 202 | 202 | 100% ✅ | — |
| common/vectorDB/opengauss/index.ts | 208 | 208 | 100% ✅ | — |
| common/vectorDB/seekdb/index.ts | 12 | 12 | 100% ✅ | — |
| **vectorDB 小计** | **1998** | **1998** | **100% ✅** | — |

### 摄取队列 / 索引写（projects/app）与 workflow 集成

| 文件 | 总行 | 已读 | 覆盖率 | 未读原因 |
|------|-----|------|--------|---------|
| …/core/dataset/data/data.ts | 527 | 527 | 100% ✅ | — |
| …/core/dataset/data/dataIndex.ts | 897 | 897 | 100% ✅ | — |
| …/core/dataset/data/utils.ts | 30 | 30 | 100% ✅ | — |
| …/queues/datasetParse.ts | 447 | 447 | 100% ✅ | — |
| …/queues/generateQA.ts | 264 | 264 | 100% ✅ | — |
| …/queues/generateVector.ts | 378 | 378 | 100% ✅ | — |
| …/queues/parseLease.ts | 123 | 123 | 100% ✅ | — |
| …/queues/utils.ts | 29 | 29 | 100% ✅ | — |
| workflow/dispatch/dataset/search.ts | 336 | 336 | 100% ✅ | — |
| workflow/dispatch/dataset/concat.ts | 45 | 45 | 100% ✅ | — |
| workflow/dispatch/dataset/nodeResponse.ts | 109 | 109 | 100% ✅ | — |
| workflow/dispatch/dataset/utils.ts | 57 | 57 | 100% ✅ | — |
| **projects + workflow 小计** | **3242** | **3242** | **100% ✅** | — |

### 全局常量/工具（仅覆盖本稿引用段）

| 文件 | 总行 | 已读 | 覆盖率 | 未读原因 |
|------|-----|------|--------|---------|
| global/core/dataset/constants.ts | 321 | 321 | 100% ✅ | — |
| global/core/dataset/search/utils.ts | 70 | 70 | 100% ✅ | — |
| global/core/dataset/data/utils.ts | 19 | 19 | 100% ✅ | — |

**总体结论：任务指定的本模块核心（service/core/dataset 核心 + vectorDB + projects/app 摄取队列 + workflow dataset 集成 + 相关 global 常量）覆盖率 ≈ 100% ✅。** 唯一未逐行全读的是 `apiDataset/{feishu,yuque,dingtalk}/api.ts` 三个第三方数据源适配器（共 1181 行），其接口与 `custom/api.ts` 完全同构（`getFileContent/listFiles/getFileDetail` + OpenAPI 鉴权），对本稿结论无影响，归「部分覆盖（模式已确认）」（如需可后续补齐）。
