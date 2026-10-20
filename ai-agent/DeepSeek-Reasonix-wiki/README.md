---
title: Reasonix 项目文档
description: 面向使用者的 Reasonix（DeepSeek-Reasonix 2.x）文档：定位与能力、四形态架构、回合循环、上下文与记忆、工具与安全边界、会话与数据、扩展体系、交付与工程门禁。
type: 项目 Wiki
---

# Reasonix

Reasonix 是一个开源的本地优先 AI 编程助手：程序跑在使用者自己的机器上，模型服务由使用者自行配置，助手在选定的项目目录里读代码、改文件、跑命令与测试。同一套内核同时支撑四种使用形态——桌面应用 Studio、终端界面、本机浏览器界面，以及编辑器通过 ACP 协议接入。

这份文档写给不需要打开源码的读者（技术选型者、新加入的成员、跨团队同事），内容覆盖它是什么、怎么运转、边界在哪，以及它把哪些架构约束变成了可执行的检查。

**本文档依据**：`v2.37.0`（`e17ef45e3`，2026-10-11）

## 页面索引

| 页面 | 内容 | 重要度 |
|------|------|--------|
| [overview.md](overview.md) | 是什么、解决什么问题、给谁用、技术栈、现状与边界 | 高 |
| [architecture.md](architecture.md) | 分层与模块地图、单点控制器、一次请求的链路 | 高 |
| [agent-loop.md](agent-loop.md) | 一个用户回合的完整生命周期、宿主拥有的失败归因 | 高 |
| [context-memory.md](context-memory.md) | 前缀缓存纪律、指令与记忆分层、压缩与投影 | 高 |
| [tools-and-safety.md](tools-and-safety.md) | 工具契约、权限档位、三维沙箱、命令安全 | 高 |
| [sessions-and-data.md](sessions-and-data.md) | 会话状态、检查点与回退、存储布局与迁移 | 中 |
| [extensions.md](extensions.md) | MCP、Extension Protocol、插件包、Skill 四条扩展路径 | 中 |
| [interfaces.md](interfaces.md) | Studio / TUI / Web / ACP 四种形态与共用通道 | 中 |
| [platform.md](platform.md) | 远端工作区、LSP、浏览器与桌面控制、安装布局 | 中 |
| [deployment.md](deployment.md) | 构建、分发、签名与更新事务 | 中 |
| [engineering.md](engineering.md) | 机械化门禁、效果测试与已知盲区 | 中 |

## 建议阅读顺序

1. [overview.md](overview.md) —— 先建立整体印象，判断它是否解决你的问题
2. [architecture.md](architecture.md) —— 再看系统怎么分层、请求怎么流过去
3. 想理解它"为什么这样设计"：接着读 [agent-loop.md](agent-loop.md) 与 [context-memory.md](context-memory.md)
4. 关心安全与可回退：读 [tools-and-safety.md](tools-and-safety.md) 与 [sessions-and-data.md](sessions-and-data.md)
5. 想扩展它：读 [extensions.md](extensions.md)
6. 要接入或部署：读 [interfaces.md](interfaces.md)、[deployment.md](deployment.md)
7. 想借鉴其工程方法：读 [engineering.md](engineering.md)

## 现状与边界

- 本文档描述的是 **2.x 线**（`studio` 分支，发布标签 `studio-v2.*`）。项目同时维护 **1.x 线**（`main-v2` 分支），两者架构不同，1.x 只接收缺陷与兼容性修复。
- 2.x 自述仍处于快速迭代阶段；会话与配置格式**尚未冻结为 2.x 的稳定契约**，跨版本迁移路径仍在补齐。
- 稳定版发布目前每个标签都会直接更新分发渠道；计划中的"每周发布列车"与双更新通道尚未落地。
- 沙箱的三个保护维度（写、读、能力）在 macOS 与 Linux 上分别有落地机制，**Windows 目前三个维度都不强制**；macOS 上经由 Apple Events / Mach 服务的宿主代为执行尚未纳入治理，且未做过实测验证。
- 本文档以 `v2.37.0` 的源码为准。项目内部分文档与实际实现存在偏差，遇到冲突时以源码为准。
