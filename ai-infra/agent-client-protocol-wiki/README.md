---
type: 项目 Wiki
title: Agent Client Protocol（ACP）项目文档
description: 面向使用者与团队成员的 Agent Client Protocol 项目 Wiki：ACP 是什么、编辑器与编码 Agent 如何用 JSON-RPC 双向对话、会话与工具调用如何运转、Rust 类型如何生成 JSON Schema 与文档、协议如何做版本演进与 RFD 治理、生态与仓库工程体系。
---

# Agent Client Protocol（ACP）

Agent Client Protocol 是一套开放协议，规定**代码编辑器**与**编码 Agent**（能自主改写代码的 AI 程序）之间怎么对话。它把「用户坐在编辑器里，让 Agent 帮忙改代码」这件事拆成两份角色清晰的实现：编辑器负责界面、文件系统、终端与用户授权，Agent 负责模型推理与工具执行，两边通过 JSON-RPC 2.0 双向通信。对使用者的直接价值是**解耦**：编辑器接一次协议就能连上所有 ACP Agent，Agent 实现一次协议就能被所有 ACP 编辑器复用。

本仓库（`agent-client-protocol/agent-client-protocol`）是这份协议的**定义与文档仓库**，不是运行时 SDK：它持有 Rust 类型的协议权威定义、由类型生成的四套 JSON Schema 产物、规范正文与 RFD 提案，以及把这些串成一条自动流水线的生成器与门禁。真正的运行时库（`agent-client-protocol` crate）与各语言 SDK 在各自仓库中维护。

**本文档依据**：`v1.11.0`（`4cf3dd8`，2026-10-09）

## 页面索引

| 页面 | 内容 | 重要度 |
|------|------|--------|
| [overview.md](overview.md) | 这个协议解决什么问题、给谁用、核心能力与技术栈、与 MCP/LSP 的关系 | 高 |
| [protocol-model.md](protocol-model.md) | 两个角色与消息模型：请求/响应/通知、方法清单、能力协商、扩展机制 | 高 |
| [session-lifecycle.md](session-lifecycle.md) | 一次协作的完整时间轴：初始化、会话建立与恢复、一次 prompt 回合、取消与关闭 | 高 |
| [tools-and-permissions.md](tools-and-permissions.md) | Agent 如何借用编辑器的文件、终端与用户授权；工具调用如何汇报；信任边界在哪 | 高 |
| [schema-pipeline.md](schema-pipeline.md) | Rust 类型如何成为唯一事实源，生成 JSON Schema、方法元数据与文档，并被 CI 锁住不漂移 | 高 |
| [versioning-and-governance.md](versioning-and-governance.md) | 三层版本解耦、v1 到 v2 的演进、RFD 流程与发布流水线 | 中 |
| [ecosystem.md](ecosystem.md) | 谁在实现这套协议、官方 SDK、Agent 注册表与合规测试工具 | 中 |
| [engineering.md](engineering.md) | 仓库结构、构建与生成命令、质量门禁、文档站与发版机制 | 中 |

## 建议阅读顺序

1. [overview.md](overview.md) —— 先判断这套协议是否解决你手上的问题
2. [protocol-model.md](protocol-model.md) —— 再看协议长什么样：谁可以调谁、能力怎么协商
3. [session-lifecycle.md](session-lifecycle.md) —— 然后跟着一次真实使用走到结束
4. [tools-and-permissions.md](tools-and-permissions.md) —— 接着看 Agent 能对机器做什么、由谁批准（**做安全评估建议不要跳过**）

按目的分叉：

- **评估是否接入 / 选型** → [overview.md](overview.md) 的对比与「现状与边界」，再扫一遍 [ecosystem.md](ecosystem.md)
- **要写一个 Agent 或 Client** → [protocol-model.md](protocol-model.md) + [session-lifecycle.md](session-lifecycle.md) + [tools-and-permissions.md](tools-and-permissions.md)
- **要基于 Schema 生成 SDK 或做代码生成** → [schema-pipeline.md](schema-pipeline.md) + [versioning-and-governance.md](versioning-and-governance.md)
- **准备参与协议设计与贡献** → [versioning-and-governance.md](versioning-and-governance.md) + [engineering.md](engineering.md)

## 现状与边界

选型前需要知道的硬事实：

- **当前稳定协议版本是整数 `1`；版本 `2` 是公开草案**。v2 的类型只在启用 `unstable_protocol_v2` 特性时编译，官方明确说明草案期仍会变更，并要求实现方同时保留 v1（v1-only 的对端会长期存在）。
- **本仓库不提供运行时**。协议类型 crate 只含类型与序列化；能直接跑的 Agent/Client 运行时、各语言 SDK 与示例都在其他仓库，本仓库里没有可直接运行的 agent 进程。
- **远程传输尚未成为协议的一部分**。标准传输目前只有 stdio（编辑器把 Agent 作为子进程启动）；Streamable HTTP / WebSocket 仍在独立 RFD 中讨论，官方文档把「远程 Agent 的完整支持」标注为进行中的工作。
- **能力协商是「缺省即不支持」**。任何未在 `initialize` 中声明的能力都必须按不支持处理；新功能默认关闭，这既是它的兼容性保障，也意味着实现方必须显式列出自己支持什么。
- **官方测试工具是实验性的**。合规测试套件（TCK）不在本仓库，且官方声明它不覆盖完整协议、通过测试不等于官方合规认定。
- **治理是 BDFL 结构**。最终决定权在 Zed 团队的项目负责人；外部提案走 RFD 与 Discussions，仓库内的「已接受协议变更」Issue 模板明确限定内部人员使用。
- **文档密度不均衡**：v1 的协议正文、v2 的迁移指南、RFD 与公告都相当完整（含 47 份 RFD、21 篇公告），但 `protocol/v1/error.mdx` 这类页面仍是「文档待补」的占位状态。
