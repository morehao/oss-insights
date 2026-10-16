---
type: 项目 Wiki
title: Model Context Protocol（MCP）项目文档
description: 面向使用者与团队成员的 Model Context Protocol 项目 Wiki：MCP 是什么、客户端与服务器如何用 JSON-RPC 对话、工具/资源/提示三原语如何工作、无状态化与版本协商怎么演进、授权与治理如何运转、仓库与 Schema 生成管线长什么样。
---

# Model Context Protocol（MCP）

Model Context Protocol 是一套开放协议，规定 **AI 应用（客户端）** 与 **提供上下文与能力的服务（服务器）** 之间怎么对话。它把「让模型用上外部数据与工具」这件事从各家自定义的插件接口里抽出来，变成一份可被多语言实现的公共协议：客户端负责承载模型与用户交互，服务器负责提供工具、数据与提示模板，两边通过 JSON-RPC 2.0 交换消息。对使用者的直接价值是 **一次接入、处处复用**：服务器实现一次协议就能被所有 MCP 客户端调用，客户端接一次协议就能连上所有 MCP 服务器。

本仓库（`modelcontextprotocol/modelcontextprotocol`）是这套协议的 **规范与文档仓库**：它持有按日期分版的协议正文（`docs/specification/<版本>/`）、以 TypeScript 类型定义的协议 Schema、由 Schema 生成的 JSON Schema 与文档产物，以及 SEP（规范增强提案）、治理文件与生成/校验脚本。真正的运行时 SDK 由各语言官方仓库分别维护，本仓库不提供可直接运行的客户端或服务器。

**本文档依据**：`2026-07-28`（`5f5440bb`，2026-07-28）

## 页面索引

| 页面 | 内容 | 重要度 |
|------|------|--------|
| [overview.md](overview.md) | MCP 解决什么问题、给谁用、核心能力、技术栈与生态位 | 高 |
| [architecture.md](architecture.md) | 客户端—主机—服务器三角色、能力协商、设计原则 | 高 |
| [protocol-basics.md](protocol-basics.md) | JSON-RPC 2.0 消息模型、请求/响应/通知、`_meta` 与公共字段 | 高 |
| [versioning-and-compatibility.md](versioning-and-compatibility.md) | 按日期分版、逐请求版本协商、Modern/Legacy/Dual 三术语与兼容策略 | 高 |
| [transports.md](transports.md) | stdio 与 Streamable HTTP 两种标准传输、自定义传输与请求元数据 | 高 |
| [server-features.md](server-features.md) | 工具、资源、提示三原语，以及 `server/discover` 自描述 | 高 |
| [client-features.md](client-features.md) | 引导（elicitation）、采样（sampling）、根目录（roots）与三者弃用状态 | 高 |
| [message-patterns.md](message-patterns.md) | 多轮往返（MRTR）、订阅、进度与取消四类交互模式 | 高 |
| [authorization.md](authorization.md) | OAuth 2.1 授权、服务器发现、三种客户端注册机制与安全要求 | 中 |
| [governance-and-lifecycle.md](governance-and-lifecycle.md) | SEP 流程、功能生命周期、工作组治理与路线图 | 中 |
| [ecosystem-and-extensions.md](ecosystem-and-extensions.md) | 扩展机制、官方扩展仓库与服务器注册表 | 中 |
| [repository-and-schema.md](repository-and-schema.md) | 仓库结构、Schema 单一事实源与生成/校验管线 | 中 |

## 建议阅读顺序

1. [overview.md](overview.md) —— 先判断这套协议是否解决你手上的问题
2. [architecture.md](architecture.md) —— 再看谁和谁说话、能力怎么协商
3. [protocol-basics.md](protocol-basics.md) —— 然后看消息本身长什么样
4. [versioning-and-compatibility.md](versioning-and-compatibility.md) —— 理解「按日期分版 + 无握手」这一关键转折
5. [transports.md](transports.md) —— 消息如何真正落到线缆上

按目的分叉：

- **评估是否接入 / 选型** → [overview.md](overview.md) 的能力清单与「现状与边界」，再扫一眼 [ecosystem-and-extensions.md](ecosystem-and-extensions.md)
- **要写一个服务器** → [server-features.md](server-features.md) + [message-patterns.md](message-patterns.md) + [transports.md](transports.md)
- **要写一个客户端** → [client-features.md](client-features.md) + [message-patterns.md](message-patterns.md) + [versioning-and-compatibility.md](versioning-and-compatibility.md)
- **要做远程 / 多租户接入** → [authorization.md](authorization.md) + [transports.md](transports.md) 的 Streamable HTTP 一节
- **要参与协议设计或贡献** → [governance-and-lifecycle.md](governance-and-lifecycle.md) + [repository-and-schema.md](repository-and-schema.md)

## 现状与边界

选型前需要知道的硬事实（均以本文档基线 `2026-07-28` 为准）：

- **`2026-07-28` 是一次破坏性重置，而非小版本叠加**。该版移除了 `initialize` 握手与协议级会话，改为「每个请求自带协议版本与能力」的无状态模型，并新增了必须实现的 `server/discover`。面向 `2025-11-25` 及更早版本编写的客户端/服务器无法直接与新版互通，需要按兼容层处理。
- **协议按日期分版，同时存在多个版本目录**。仓库内并排保存 `2024-11-05`、`2025-03-26`、`2025-06-18`、`2025-11-25`、`2026-07-28` 与 `draft`，文档站默认展示最新版，历史版本仍可按需查阅。
- **本仓库不提供运行时**。它只含协议定义、Schema、文档与工具脚本；能直接跑的 SDK 在各语言官方仓库，本仓库里没有可启动的客户端或服务器进程。
- **授权是可选能力**。基于 HTTP 的传输应当遵循 OAuth 2.1 授权；stdio 传输通常从环境变量取凭证，不应走完整授权流程。是否需要授权取决于你的部署形态。
- **部分能力已进入弃用期**。`2026-07-28` 将日志（logging）、采样（sampling）、根目录（roots）标记为弃用，约定至少在规范中保留一段时间供迁移，新实现不应再采用。
- **许可正在迁移**。仓库正从 MIT 转向 Apache-2.0：新增代码与规范贡献采用 Apache-2.0，文档贡献（不含规范本体）采用 CC-BY-4.0，历史上以 MIT 提交且未明确同意再许可的部分仍保留 MIT。
