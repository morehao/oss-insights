---
title: MOVO 项目文档
description: 面向使用者的 MOVO 项目 Wiki：基于 DeepSeek Harness（DSH）的自托管企业级 Agent 平台，覆盖定位与能力、12 服务架构、DSH 内核宿主化、会话与一轮执行、模型接入与凭据、Skill/工具/MCP、企业知识与研究、内容与文件生成、管理治理、部署运维，以及现状与选型边界。
type: 项目 Wiki
---

# MOVO

MOVO 是一个可私有化部署的企业级 Agent 平台（Community Edition）。它把 DeepSeek Harness（DSH）这个 Agent 内核装进一个独立进程，再围绕它补齐一个组织真正上线需要的东西：统一入口的用户工作台与管理后台、企业知识与文档处理、Skill/工具/MCP 等能力接入、身份与权限、审批与审计、内容与文件交付，以及一套 `docker compose` 就能起来的整套服务。

一句话理解：**DSH 负责"智能体怎么运行"，MOVO 负责"智能体怎么进入企业生产环境"。**

本文档面向不打开源码的读者——技术选型者、新加入的成员、跨团队同事。读完能回答"这是什么、怎么运转、怎么用、怎么部署、值不值得选"。

**本文档依据**：`v0.2.3`（`500612c`，2026-10-09）

## 页面索引

| 页面 | 内容 | 重要度 |
|------|------|--------|
| [overview.md](overview.md) | 项目是什么、解决什么问题、给谁用、核心能力与技术栈 | 高 |
| [architecture.md](architecture.md) | 12 个服务怎么组成、双网络分区、网关路由、一次请求的主链路与关键取舍 | 高 |
| [dsh-runtime-host.md](dsh-runtime-host.md) | 为什么把 DSH 关进独立进程、kernel 契约、宿主职责、模型/工具如何回调回业务侧 | 高 |
| [conversation-lifecycle.md](conversation-lifecycle.md) | 一条会话与一轮执行的生命周期：状态、事件双轨、持久化、断线恢复、取消、会话共享 | 高 |
| [model-and-credentials.md](model-and-credentials.md) | 模型中心与 Runtime Profile 编译、模型网关反向代理、短期令牌与凭据租约 | 高 |
| [skills-tools-mcp.md](skills-tools-mcp.md) | Skill 体系、工具与 MCP 接入、插件、审批与执行回执 | 高 |
| [knowledge-and-research.md](knowledge-and-research.md) | 文档接入与解析、分块向量化、检索与引用、多源研究与证据 | 高 |
| [content-generation.md](content-generation.md) | 报告 / PPTX / XLSX / PDF / Word 的生成、质量闭环与文件交付 | 中 |
| [administration-and-governance.md](administration-and-governance.md) | 首次初始化、组织/用户/岗位能力、多租户、配额、审计与平台配置 | 高 |
| [deployment-and-operations.md](deployment-and-operations.md) | Compose 部署形态、`./movo` CLI、配置、升级备份恢复、生产基线 | 高 |
| [limits-and-selection.md](limits-and-selection.md) | 能力边界、社区版限制、许可证、单机基线、与 DSH/同类对比等硬事实 | 高 |

## 建议阅读顺序

1. [overview.md](overview.md) —— 先建立整体印象：它解决什么问题、由哪些能力组成。
2. [architecture.md](architecture.md) —— 再看系统怎么组织：有哪些服务、请求怎么流过去。
3. [dsh-runtime-host.md](dsh-runtime-host.md) —— 理解它和 DSH 的边界：内核为什么被单独关起来、怎么通信。
4. [conversation-lifecycle.md](conversation-lifecycle.md) —— 顺着"一次对话"看完整链路：一轮怎么跑、掉线怎么恢复。
5. [model-and-credentials.md](model-and-credentials.md) 与 [skills-tools-mcp.md](skills-tools-mcp.md) —— 理解模型与能力如何被安全地配给内核。
6. [knowledge-and-research.md](knowledge-and-research.md) 与 [content-generation.md](content-generation.md) —— 理解"知识进上下文、交付物出平台"这条业务主线。

按读者目的分叉：

- 想尽快跑起来 → 直接看 [deployment-and-operations.md](deployment-and-operations.md)；
- 负责平台管理 → 看 [administration-and-governance.md](administration-and-governance.md)；
- 正在做选型 → 先读 [limits-and-selection.md](limits-and-selection.md)，再回看 [overview.md](overview.md)。

## 现状与边界

MOVO 把 DSH 做成一个"可交付的产品"，这个定位本身决定了它的边界（详见 [limits-and-selection.md](limits-and-selection.md)）：

- **不适合只需要底层 Agent Runtime 的场景**。如果目标只是跑一个 Agent 内核，DSH 本身就够，MOVO 的价值在它补上的产品与基础设施层。
- **浏览器 Agent 与 Code Agent 不在本仓库**。这两项依赖独立分发的专有桌面客户端 MOVO Desktop，源码不在社区版内；自托管 Web 端只覆盖对话、研究、知识、文件和内容生成。
- **默认是单机基线**。官方 Compose 面向单机自托管，高可用数据库、外部对象存储、集中式密钥与滚动升级需自行替换验证。
- **许可证是"源码可见"而非 OSI 开源**。MOVO Community License 以 Apache 2.0 为基础附加了限制（禁止未经授权做托管式多租户 SaaS、禁止移除 Logo 等），商业授权需联系官方。
- **DSH 仍是快速演进的研究/开发者产品**。MOVO 通过锁版本的方式把它当作依赖，内核行为以对应版本为准。
