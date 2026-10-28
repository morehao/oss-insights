---
title: Hindsight 项目文档
description: 面向使用者的 Hindsight（agent memory system）文档：核心概念、retain/recall/reflect 三大操作、后台学习、存储与部署，帮助不读源码的读者理解并接入这套智能体长期记忆系统。
type: 项目 Wiki
---

# Hindsight

Hindsight 是一套给 AI agent 用的**长期记忆系统**。它不做简单的对话历史回放，而是把 agent 见过的事实、经历、以及由此形成的判断组织成可检索、可演化的记忆，让 agent 越用越"懂"。它既是一个可自部署的服务（REST API + MCP），也提供 Python / Node / Go / Rust 客户端与嵌入式运行方式。

**本文档依据**：`v1.1.0`（`a5a619b5c`，2026-10-21）

## 页面索引

| 页面 | 内容 | 重要度 |
|------|------|--------|
| [overview.md](overview.md) | 这个项目解决什么问题、给谁用、技术栈与同类差异 | 高 |
| [architecture.md](architecture.md) | 系统分成哪几层、模块地图、一次请求的主链路 | 高 |
| [memory-model.md](memory-model.md) | 核心概念：bank、三类事实、实体与链接、observation、mental model | 高 |
| [retain.md](retain.md) | 写入链路：从原始文本到可检索的记忆单元 | 高 |
| [recall.md](recall.md) | 检索链路：四路召回、融合、重排与评分 | 高 |
| [reflect.md](reflect.md) | 反思操作：agent 如何用工具循环给出深度回答（含决策编排形态） | 中 |
| [learning.md](learning.md) | 后台学习：观察固化（决策模型分流）、mental model 刷新、knowledge pages、变更历史 | 高 |
| [storage.md](storage.md) | 数据存储：多后端抽象、核心表、向量与全文索引 | 中 |
| [integration.md](integration.md) | 客户端 SDK、CLI、MCP、框架与编码代理集成、Control Plane | 中 |
| [deployment.md](deployment.md) | 部署形态、配置层级、后台 worker、可观测性与安全 | 中 |

## 建议阅读顺序

1. [overview.md](overview.md) —— 先建立整体印象：它是什么、解决什么问题、代价是什么
2. [architecture.md](architecture.md) —— 再看系统怎么组织、请求怎么流过去
3. [memory-model.md](memory-model.md) —— 掌握几个核心概念，后面几页都建立在这上面
4. 按目的分叉：
   - 想理解"记忆怎么写进去" → [retain.md](retain.md)
   - 想理解"记忆怎么被检索出来" → [recall.md](recall.md)
   - 想理解"agent 怎么做深度推理" → [reflect.md](reflect.md)
   - 想理解"记忆怎么自己变聪明" → [learning.md](learning.md)
   - 关心数据怎么落地、怎么选存储 → [storage.md](storage.md)
   - 只想知道怎么接进自己的应用 → [integration.md](integration.md)
   - 要部署和运维 → [deployment.md](deployment.md)

## 现状与边界

- **本文档基于发布版 `v1.1.0`**（2026-10-21）。晚于该版本的变更未涵盖；上游默认分支上的改动不在文档范围内。
- **Hindsight Cloud 不在覆盖范围内**：托管版（`api.hindsight.vectorize.io`）是闭源服务，本文档只描述本仓库中的开源代码。
- **默认数据库是 PostgreSQL**：项目同时支持 Oracle AI Database，但 Oracle 的支持依赖商用数据库与对应驱动，本文档对 Oracle 只做说明、不铺开细节。
- **基准测试成绩来自项目自述**：仓库的 README 中关于 LongMemEval 等指标的结论由项目方发布，本文档不复述具体数字。
- **各集成包的成熟度不一致**：仓库内集成包数量很多，覆盖面差异较大，本文档只描述其组织方式而非逐个背书。
- 文档面向"不打开源码的读者"，文中不逐句标注源码位置；每页末尾的「依据」行给出该页结论对应的源码文件，供想深究的人自行核对。

> 依据：`README.md:22-52`、`pyproject.toml:1-9`
