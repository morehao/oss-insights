---
title: EverOS 项目文档
description: 面向使用者的 EverOS 文档：Markdown 本地优先的 AI Agent 长期记忆框架，覆盖八类记忆模型、写入与离线派生、四种检索方法、知识库、三件套存储与一致性、部署运维，帮助不读源码的读者理解并接入。
type: 项目 Wiki
---

# EverOS

EverOS 是一套**本地优先的 AI Agent 长期记忆框架**：它把对话、文件、Agent 轨迹沉淀成**人类可读、可编辑、可 Git 版本化**的 Markdown 文件，再用一层本地 SQLite 与向量/全文索引支撑快速检索和"越用越懂"的离线演化。它既是一个可自托管的 HTTP 服务，也是一组 Python 库；目标是让单个开发者或小团队在自己的机器上获得一套可带走、可审计的记忆底座，而不必依赖托管数据库。

**本文档依据**：`v1.5.1`（`5b0979b`，2026-10-22）

## 页面索引

| 页面 | 内容 | 重要度 |
|------|------|--------|
| [overview.md](overview.md) | 解决什么问题、给谁用、核心能力、技术栈与同类差异 | 高 |
| [architecture.md](architecture.md) | DDD 四层与横切、依赖方向、模块地图、一次请求的主链路 | 高 |
| [memory-model.md](memory-model.md) | 八类记忆、三种落盘策略、user/agent 双轨、作用域划分 | 高 |
| [write-path.md](write-path.md) | 消息如何变成记忆：缓冲→边界→抽取→落 md→OME 异步派生与 reflection | 高 |
| [search-and-get.md](search-and-get.md) | 四种检索方法、hybrid 融合的取舍、`/get` 精确取回 | 高 |
| [knowledge.md](knowledge.md) | 文档→LLM 主题树的三层知识库、存储布局与 CRUD | 中 |
| [storage-and-consistency.md](storage-and-consistency.md) | md/SQLite/派生索引三件套、目录布局、cascade 守护、一致性模型 | 高 |
| [deployment.md](deployment.md) | 安装与 init、配置优先级、CLI、健康探针与运维 | 中 |

## 建议阅读顺序

1. [overview.md](overview.md) —— 先建立整体印象，判断它是否解决你的问题。
2. [architecture.md](architecture.md) —— 再看系统怎么分层组织，建立全局坐标。
3. [memory-model.md](memory-model.md) —— 理解"记忆到底有哪几种、存在哪"。
4. [write-path.md](write-path.md) → [search-and-get.md](search-and-get.md) —— 跟着一次写入与一次检索走完主链路。
5. 按需深入：[knowledge.md](knowledge.md)（要把文档变成可检索库时）、[storage-and-consistency.md](storage-and-consistency.md)（关心可靠性与数据安全时）、[deployment.md](deployment.md)（要自己跑起来时）。

## 现状与边界

- **定位是单机 / 小团队**：官方明确把多租户、端到端云同步、分布式部署列为 v1 之外（后续版本规划），当前不提供这些能力。文档描述的是本地单进程形态。
- **写入强一致、读取最终一致**：`/flush` 返回 `extracted` 时 episode 的 Markdown 已落盘，但检索所依赖的派生索引会滞后（cascade 处理时间，通常亚秒级，负载下可能到十几秒）。刚写入就立刻检索可能查不到，需要重试。
- **派生索引无降级搜索兜底**：当向量/全文索引不可用时，没有"直接 grep Markdown"的回退搜索路径，正确做法是从 Markdown 重建索引。
- **多数记忆种类异步产生**：atomic facts / foresight / profile / agent case / skill 由离线引擎（OME）在抽取之后异步派生，`/flush` 刚返回时通常还查不到，需要等策略跑完。
- **reflection 默认关闭**：离线记忆合并（reflection）默认不启用，需在 `ome.toml` 里手动打开。
- **许可证**：Apache License 2.0（见仓库 `LICENSE`）。
