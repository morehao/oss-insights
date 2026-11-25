---
title: LoopForge 项目文档
description: 面向使用者的 LoopForge 项目 Wiki：自托管的编码 Agent 交付平台，覆盖定位与两条工作路径、进程与模块架构、SOP 引擎、对话工作台、角色与执行器、沙箱与执行机器、需求与交付、HTTP/MCP 接口、定时任务与通知、工程洞察与成本，以及部署与能力边界。
type: 项目 Wiki
---

# LoopForge

LoopForge 是一个**自托管的编码 Agent 交付平台**：团队成员在对话里说清要做什么，Main Bot 就拉起一个 Docker 沙箱、检出仓库、分派给不同项目角色，最后开出一个 PR。当某种协作方式被证明有效，可以把它固化成一条 **SOP**（Agent、脚本、人工审批节点组成的工作流），让下一个需求按同样的步骤、同样的检查、同样的签核来交付。

一句话理解：**编码 Agent 负责"把活干出来"，LoopForge 负责"谁在什么环境下、按什么顺序、经过什么检查、最后由谁签核把它交付出去"。**

它不替换 Claude Code、Codex、CodeBuddy 这些编码 CLI，而是把十个这样的 CLI 编排进一个团队可用的流程里，全部跑在你自己的基础设施上（Docker Compose，MIT 协议）。

本文档面向不打开源码的读者——技术选型者、新加入的成员、跨团队同事。读完能回答"这是什么、怎么运转、怎么用、怎么部署、值不值得选"。

**本文档依据**：`main`（`2ad5659`，2026-11-25）。该快照位于 `v0.4.2` 之后 17 个提交，文档按此工作树撰写。

## 页面索引

| 页面 | 内容 | 重要度 |
|------|------|--------|
| [overview.md](overview.md) | 项目是什么、解决什么问题、两条工作路径、核心能力表、技术栈、与同类对比 | 高 |
| [architecture.md](architecture.md) | 进程角色（api/worker/all/runner）、模块分层、一次请求的主链路、代码库总览 | 高 |
| [sop-engine.md](sop-engine.md) | SOP 图模型、节点与边、校验门与重做边、调度、并行分支、租约、断点恢复 | 高 |
| [conversation-workbench.md](conversation-workbench.md) | Main Bot 对话、会话工作区、沙箱绑定与快照、消息排队与打断、产物发布 | 高 |
| [roles-and-executors.md](roles-and-executors.md) | 角色（prompt/executor/model/rules/Skill/MCP）、十个执行 CLI、模型与凭证 | 高 |
| [sandboxes-and-runners.md](sandboxes-and-runners.md) | Docker/native 沙箱、平台 worker 与执行机器、Runner 协议、仓库缓存 | 中 |
| [requirements-and-delivery.md](requirements-and-delivery.md) | 需求→任务→运行→交付、PR/MR 跟踪、直推分支、多仓库 | 高 |
| [mcp-and-api.md](mcp-and-api.md) | 平台 HTTP/OpenAPI API、平台 MCP server、结构化 gate MCP、外部集成 | 中 |
| [scheduling-and-notifications.md](scheduling-and-notifications.md) | AI 定时任务、共享 SOP 调度、需求自动派发、通知渠道 | 中 |
| [insights-and-cost.md](insights-and-cost.md) | 工程指标、业务域报告、Token/成本核算、会话用量口径 | 中 |
| [deployment-and-boundaries.md](deployment-and-boundaries.md) | Compose 部署、账户与访问模型、升级备份、能力边界与选型硬事实 | 高 |

## 建议阅读顺序

1. [overview.md](overview.md) —— 先建立整体印象：它解决什么问题、由哪些能力组成、和别的工具怎么区分。
2. [architecture.md](architecture.md) —— 再看系统怎么组织：进程角色、模块分层、一次请求怎么流过去。
3. 顺着"两条工作路径"各读一页：[sop-engine.md](sop-engine.md)（结构化交付）与 [conversation-workbench.md](conversation-workbench.md)（对话式交付）。
4. [roles-and-executors.md](roles-and-executors.md) 与 [sandboxes-and-runners.md](sandboxes-and-runners.md) —— 理解"谁来做"和"代码在哪里跑"。
5. [requirements-and-delivery.md](requirements-and-delivery.md) —— 理解这一切如何汇到"需求→合入的代码"这条主线上。
6. 按需展开：[mcp-and-api.md](mcp-and-api.md)、[scheduling-and-notifications.md](scheduling-and-notifications.md)、[insights-and-cost.md](insights-and-cost.md)。

按读者目的分叉：

- 想尽快跑起来 → 直接看 [deployment-and-boundaries.md](deployment-and-boundaries.md)；
- 正在做选型 → 先读 [overview.md](overview.md) 的对比表，再读 [deployment-and-boundaries.md](deployment-and-boundaries.md) 的边界与硬事实；
- 想理解为什么这样设计 → [sop-engine.md](sop-engine.md) 与 [conversation-workbench.md](conversation-workbench.md)。

## 现状与边界

LoopForge 的定位决定了它的边界（详见 [deployment-and-boundaries.md](deployment-and-boundaries.md)）：

- **面向受信任的团队，不是多租户 SaaS**。默认安装监听所有网卡，登录用户可跨项目读取数据；Docker socket 权限等同于宿主 Docker 守护进程控制权，不适合把不可信多方放在同一个实例里隔离。
- **不替换编码 CLI，而是编排它们**。每个 Agent 节点、每一轮对话都由某个编码 CLI 实际执行；LoopForge 补的是它周围的角色、沙箱、检查点与历史。
- **模型用量自付**。平台按公开价目核算成本，但模型 token 由你自带 key 实际支出。
- **GitHub 是可选的**。对话、文档与新项目在任务工作区即可完成；只有要改已有仓库或开 PR 时才需要连接 GitHub（或用 Git SSH 接入 GitLab/Gitea 等）。
- **单机 Compose 是官方基线**。高可用数据库、外部对象存储、集中密钥等需自行替换验证；官方验证记录明确列出了"未完全实测"的范围（付费 Codex/Claude、外部知识服务、企业通知、真实多云池等）。
