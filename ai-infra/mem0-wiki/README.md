---
type: 项目 Wiki
title: Mem0 项目文档
description: 面向使用者与团队成员的 Mem0 项目 Wiki：Mem0 是什么、怎么给 AI Agent 加长期记忆、写入与检索两条链路怎么运转、四类 Provider 怎么扩展、数据存在哪、自托管 Server 怎么部署、TS SDK 与 CLI 及 MCP/插件生态怎么接入。
---

# Mem0

Mem0（读作 "mem-zero"）是给 AI Agent 加**长期记忆**的一层组件。它夹在应用与模型之间：应用把对话交给 Mem0，Mem0 用 LLM 把对话里的关键事实提取成一条条记忆存起来；下一次提问前，应用先向 Mem0 检索出相关记忆，再塞进模型的 prompt。这样 Agent 不必每轮重放整段对话，也能记住用户偏好、历史决策与跨会话的上下文。

它同时以三种形态对外提供：可嵌入的 **Python/TypeScript SDK**、可自部署的 **Server**（FastAPI + 自带 Dashboard）、以及托管的 **Cloud Platform**。本仓库是这三者的开源部分——SDK、自托管 Server、CLI 与各类编辑器/框架集成都在这里，Cloud Platform 的专有优化不在此仓库内。

**本文档依据**：`v2.0.17`（`12c47f52`，2026-08-05）

## 页面索引

| 页面 | 内容 | 重要度 |
|------|------|--------|
| [overview.md](overview.md) | 解决什么问题、给谁用、三种运行形态、核心能力、技术栈与边界 | 高 |
| [architecture.md](architecture.md) | 系统分层、模块地图、装配过程、add/search 主链路与关键取舍 | 高 |
| [memory-pipeline.md](memory-pipeline.md) | 写入链路：ADD-only 提取管线如何把对话变成去重后的记忆与实体 | 高 |
| [retrieval.md](retrieval.md) | 检索链路：语义 + BM25 + 实体三路信号如何打分融合 | 高 |
| [provider-system.md](provider-system.md) | 四类 Provider 扩展点：LLM / Embedding / VectorStore / Reranker 怎么接 | 中 |
| [data-model.md](data-model.md) | 记忆的字段、作用域、三处存储与生命周期 | 中 |
| [self-hosted-server.md](self-hosted-server.md) | 自托管 Server：认证、数据表、API 路由与 Docker 部署 | 中 |
| [ecosystem.md](ecosystem.md) | TS SDK、CLI、MCP 与 skills、编辑器/框架集成 | 中 |

## 建议阅读顺序

1. [overview.md](overview.md) —— 先判断 Mem0 是否解决你手上的问题、该用哪种形态
2. [architecture.md](architecture.md) —— 再看系统怎么组织、一次 add / search 经过哪些环节
3. [memory-pipeline.md](memory-pipeline.md) —— 深入写入链路，理解"记忆"是怎么被造出来的
4. [retrieval.md](retrieval.md) —— 深入检索链路，理解为什么它不只是一个向量库

按目的分叉：

- **只想快速用起来** → [overview.md](overview.md) 的三种形态对比，然后直接跳到 [ecosystem.md](ecosystem.md) 挑 SDK 或 CLI
- **要选型 / 评估** → [overview.md](overview.md) 的「现状与边界」+ [retrieval.md](retrieval.md) 的信号说明
- **要接一个自有模型或向量库** → [provider-system.md](provider-system.md)
- **要自建服务给团队用** → [self-hosted-server.md](self-hosted-server.md) + [data-model.md](data-model.md)
- **要做二次开发** → [architecture.md](architecture.md) 的模块地图与关键取舍

## 现状与边界

选型前需要知道的硬事实（以本文档基线 `v2.0.17` 为准）：

- **开源 SDK 不含 Platform 的专有优化**。README 列出的 benchmark 数字（LoCoMo / LongMemEval / BEAM）来自 Mem0 托管平台，官方明确说明开源用户应预期"方向相似但数值不同"。
- **部分能力是 Platform 专有**：Graph Memory（图记忆）只在托管平台提供，开源 SDK 的实体匹配只做"共现实体加权"，不构建知识图谱；时间推理（temporal reasoning）中用户可直接传入的 `timestamp` / `reference_date` 参数在开源 SDK 里**会直接报错**，不属于开源能力。
- **写入默认是 ADD-only**：一次 `add` 只做"新增"，不自动改写或删除旧记忆。需要纠正时要显式调用 `update` / `delete`。
- **混合检索依赖向量库与 NLP extra**：BM25 关键词检索要求所用向量库实现了 `keyword_search`；实体提取依赖 spaCy 与 `[nlp]` extra。两者不具备时能力会**降级**而非报错（退化为纯语义检索）。
- **历史记录用本地 SQLite**：`history` 与最近消息存在本地 SQLite 文件里，按进程/单机本地生效，设计上不是多实例共享的分布式存储。
- 本仓库同时是 **monorepo**：Python SDK（`mem0/`）、TypeScript SDK（`mem0-ts/`）、CLI（`cli/`）、自托管 Server（`server/`）、编辑器与框架集成（`integrations/`）各自独立发版，版本号不必一致。
