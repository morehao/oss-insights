---
title: Engram 项目文档
description: 面向使用者的 Engram（AI coding agent 持久记忆）中文项目 Wiki：单二进制 + SQLite 的本地优先记忆系统，覆盖记忆模型、MCP 工具、Git/云同步、Agent 接入与运维，帮助不读源码的读者理解并接入。
type: 项目 Wiki
---

# Engram

Engram 是给 AI 编码 Agent 用的**持久记忆系统**。Agent 在会话结束时本该忘掉一切，Engram 让它把值得留存的决策、Bug 修复、发现与约定存进本地 SQLite，下次会话再检索回来。它是一个 Go 单二进制，通过 MCP、HTTP、CLI、TUI 四种接口暴露同一套记忆，默认零依赖、纯本地，云同步是可选的团队共享层。

**本文档依据**：`v3.6.0`（`98912f8`，2026-10-24）

## 页面索引

| 页面 | 内容 | 重要度 |
|------|------|--------|
| [overview.md](overview.md) | 是什么、解决什么问题、给谁用、技术栈与同类差异、现状与边界 | 高 |
| [architecture.md](architecture.md) | 系统分层、模块地图、本地与云两套运行时、一次保存/检索的主链路 | 高 |
| [memory-model.md](memory-model.md) | 核心概念：session、observation、prompt、relation，scope、topic_key、复习生命周期、去重 | 高 |
| [mcp-tools.md](mcp-tools.md) | 27 个 MCP 工具、agent/admin 两种 profile、渐进式检索、Agent 工作流 | 高 |
| [interfaces.md](interfaces.md) | CLI 命令、本地 HTTP API、TUI 三种接口一览 | 中 |
| [sync-and-cloud.md](sync-and-cloud.md) | Git 分块同步、云 autosync、cloud serve、Postgres、仪表盘 | 中 |
| [agents-and-setup.md](agents-and-setup.md) | 14 种 Agent 接入、插件与 hooks、项目解析与工作区绑定 | 中 |
| [operations.md](operations.md) | 安装、配置与环境变量、doctor 诊断、自测、升级与回滚 | 中 |

## 建议阅读顺序

1. [overview.md](overview.md) —— 先判断这东西解不解决你的问题、代价是什么
2. [architecture.md](architecture.md) —— 建立整体结构印象：谁存数据、谁只是接口
3. [memory-model.md](memory-model.md) —— 理解记忆是怎么被组织、去重、复习的
4. 想直接用 → [agents-and-setup.md](agents-and-setup.md)（如何接入你的 Agent）→ [mcp-tools.md](mcp-tools.md)（Agent 会调用哪些工具）
5. 要看代码/脚本调用 → [interfaces.md](interfaces.md)（CLI 与 HTTP）
6. 团队共享或自建云 → [sync-and-cloud.md](sync-and-cloud.md)
7. 运维排障与升级 → [operations.md](operations.md)

## 一句话记住它

> **本地 SQLite 是唯一事实源；云只是可选的复制与共享，不是数据的所有者。**

## 现状与边界

- **不是全量记录器**：engram 依赖 Agent 主动判断"什么值得存"，不做原始工具调用的无差别捕获，也不内置压缩管线。
- **本地是单用户信任模型**：本地 HTTP / MCP 面向同一用户的受信进程，不承诺互不信任客户端或项目之间的隔离；项目级权限边界只存在于 cloud 服务器一侧。
- **云是可选且 scope 受限**：云同步需要显式 `--project` 逐个项目 enroll；插件层的自动云注册/登录/升级编排目前尚未实现，云流程走 CLI。
- **记忆复习是提示而非到期删除**：`review_after` 只是类型化的复习提醒，过期记忆仍可检索，硬过期尚未实现。
- **平台与数据目录限制**：已知 NFS / SMB 数据目录会被拒绝启动（SQLite WAL 在网络文件系统上不安全）。
