---
title: RAGFlow 项目文档
description: 面向使用者的 RAGFlow 项目 Wiki：RAG 引擎与 Agent 平台的整体架构、文档处理与摄取、检索与对话、Agent 工作流、数据源同步、存储引擎、部署运维，以及 Go 迁移现状。
type: 项目 Wiki
---

# RAGFlow

RAGFlow 是一套开源的 RAG（Retrieval-Augmented Generation，检索增强生成）引擎与 Agent 平台。它把"企业私有文档"变成大模型可以引用的知识：先解析并切分文档，建成可检索的知识库，再在对话或工作流中按问题检索出相关片段，交给大模型生成带引用的答案。围绕这条主线，它还提供可视化 Agent 工作流编排、多数据源同步、以及一整套可部署的后端服务。

本文档面向不打开源码的读者：技术选型者、新加入的成员、跨团队同事。读者读完能回答"这是什么、怎么运转、怎么用、怎么部署"。

**本文档依据**：`v1.0.0-rc1`（`14bb02ee`，2026-09-29）

> 本文档基于预发布版本 `v1.0.0-rc1` 撰写。该版本处于从 Python 到 Go 的重写中途，晚于该版本的变更未涵盖。

## 页面索引

| 页面 | 内容 | 重要度 |
|------|------|--------|
| [overview.md](overview.md) | 项目是什么、解决什么问题、给谁用、核心能力与技术栈 | 高 |
| [architecture.md](architecture.md) | 系统分层、服务拓扑、模块地图与主链路 | 高 |
| [document-processing.md](document-processing.md) | 文档解析与切分：DeepDoc、多格式解析、分块策略 | 高 |
| [ingestion.md](ingestion.md) | 数据摄取流水线：模板、组件、任务分发与进度 | 高 |
| [retrieval-and-chat.md](retrieval-and-chat.md) | 检索与对话：混合检索、重排、引用、会话 | 高 |
| [agent.md](agent.md) | Agent 与工作流：画布、组件、运行时、工具调用 | 高 |
| [data-sync.md](data-sync.md) | 数据源连接器与同步服务 | 中 |
| [storage-engines.md](storage-engines.md) | 文档检索引擎与配套存储服务 | 中 |
| [deployment.md](deployment.md) | 部署形态、运行模式、配置与运维要点 | 高 |
| [go-migration.md](go-migration.md) | Python 到 Go 的迁移现状与工程约定 | 中 |

## 建议阅读顺序

1. [overview.md](overview.md) —— 先建立整体印象：它解决什么问题、由哪些能力组成。
2. [architecture.md](architecture.md) —— 再看系统怎么组织：有哪些服务、请求怎么流过去。
3. [document-processing.md](document-processing.md) 与 [ingestion.md](ingestion.md) —— 理解"文档怎么变成可检索的知识"。
4. [retrieval-and-chat.md](retrieval-and-chat.md) —— 理解"知识怎么被检索出来并生成答案"。
5. [agent.md](agent.md) —— 理解可视化工作流与 Agent 能力。

按读者目的分叉：

- 想尽快跑起来 → 直接看 [deployment.md](deployment.md)；
- 想接入自己的数据源 → 看 [data-sync.md](data-sync.md)；
- 想了解存储与检索引擎选型 → 看 [storage-engines.md](storage-engines.md)；
- 关心工程现状与重写进度 → 看 [go-migration.md](go-migration.md)。

## 现状与边界

- **版本**：本文档描述 `v1.0.0-rc1`，是一个 release candidate（预发布版本），不是最终稳定版。
- **重写中途**：该版本处于 Python 到 Go 的重写过程中。后端主服务已是 Go 实现，但仓库内仍残留部分 Python 代码（如 Python SDK、WhatsApp 通道、部分提示词与内嵌资源），线上部署在特定配置下仍可能回落到 Python 入口。
- **各检索引擎成熟度不一**：文档检索层抽象出多种存储引擎，其中 Elasticsearch 路径完整可用；Infinity 路径在当前版本仍是等待官方 Go SDK 的占位实现。
- **文档未覆盖**：本文档不评估性能数字、容量上限与生产规模，这些在源码中未能确认，故不写入。
