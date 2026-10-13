---
title: DeepSeek Harness（dsh）项目文档
description: 面向使用者与团队的 DeepSeek Harness 项目 Wiki：项目定位、插件化架构、Agent 回合生命周期、工具体系与安全边界、会话数据、模型接入、扩展机制、运行形态与工程体系。
---

# DeepSeek Harness（`dsh`）

DeepSeek Harness 是 DeepSeek AI 开源的 agent harness（智能体运行框架）：它把「模型 + 工具 + 会话历史 + 运行环境」组装成一个可长期使用的编码/自动化助手，并同时提供 Web 图形界面、命令行一次性任务、桌面应用与多种程序化接口。它最鲜明的特征不是某个功能，而是一条架构主张——**没有特权内核，一切皆插件**：模型适配器、工具注册表、会话日志、乃至 Agent 主循环本身都是可替换的插件，通过配置而不是改源码来改变产品形态。

这套文档面向**不打算读源码的人**：技术选型者、新加入的成员、跨团队同事。它讲清「这是什么、怎么运转、为什么这样设计、代价在哪」，因此正文以散文、图和表为主，几乎不出现代码。

**本文档依据**：`dsh-v0.2.0-rc.2`（提交 `639ed01539`，即当时 `origin/master` 的顶端），2026-09-29

> 说明：这个项目自带一套相当完整的官方文档（`docs` 下有 128 份子系统文档，中英双语）与工程笔记。本 Wiki 不是它们的替代或翻译，而是一条**面向外部读者的阅读线**：先建立整体印象，再按需下钻。遇到与源码冲突之处，以源码为准。

## 页面索引

| 页面 | 内容 | 重要度 |
|------|------|--------|
| [overview.md](overview.md) | 这个项目解决什么问题、给谁用、核心能力与技术栈 | 高 |
| [architecture.md](architecture.md) | 无特权内核、插件树如何组装、capability seam、事件分域 | 高 |
| [agent-loop.md](agent-loop.md) | 一次对话回合的完整生命周期：step、提示词组装、中断与重试 | 高 |
| [tools-and-safety.md](tools-and-safety.md) | 工具体系、执行管道、审批策略、沙箱、结果外溢 | 高 |
| [sessions-and-data.md](sessions-and-data.md) | 会话事件日志、格式世代与迁移、存储与检索、附件与交付物 | 高 |
| [model-layer.md](model-layer.md) | 模型接入的抽象与适配器、流式协议、凭据与设置、类型化 RPC | 中 |
| [extensibility.md](extensibility.md) | 写插件、技能、MCP、子代理、工作流、钩子与后台作业 | 高 |
| [interfaces.md](interfaces.md) | CLI / Web / 桌面 / SDK / ACP 五种运行形态与安装运行 | 中 |
| [engineering.md](engineering.md) | 仓库结构、双聚合类型体系、质量门禁与文档工程 | 中 |

## 建议阅读顺序

1. [overview.md](overview.md) —— 先建立整体印象，判断这东西是否解决你的问题
2. [architecture.md](architecture.md) —— 再看系统怎么组织，理解「一切皆插件」到底指什么
3. [agent-loop.md](agent-loop.md) —— 然后看一次请求在系统里怎么走完
4. [tools-and-safety.md](tools-and-safety.md) —— 接着看它能对机器做什么、边界在哪里（**选型者建议不要跳过这页**）
5. [sessions-and-data.md](sessions-and-data.md) —— 数据长什么样、存在哪、能不能搬走

按目的分叉：

- **只想用起来** → 直接看 [interfaces.md](interfaces.md) 的安装与运行，再回 [tools-and-safety.md](tools-and-safety.md) 了解权限设置
- **想写插件或接入自有模型** → [extensibility.md](extensibility.md) + [model-layer.md](model-layer.md)
- **想评估工程可维护性 / 准备长期跟进** → [engineering.md](engineering.md) + [sessions-and-data.md](sessions-and-data.md) 的格式演进一节
- **要做安全评估** → [tools-and-safety.md](tools-and-safety.md) + [overview.md](overview.md) 的「现状与边界」

## 现状与边界

选型前需要知道的硬事实，来自仓库自身的声明与约束：

- **开发者预览阶段，官方明示会有破坏兼容性的变更。** 公共 API 处于 pre-stable，升级时需要同步更新所有消费方。
- **没有经过安全审计，不应被当作安全边界。** 沙箱、审批与权限控制只能降低风险，不能保证隔离；项目会执行模型生成的代码与命令、加载第三方插件、访问网络与文件。官方建议在最小权限、可丢弃的环境中使用，并对可访问的文件做好备份。
- **官方现阶段不接受外部代码贡献（Pull Request）。** 参与方式是通过 Discussions 反馈问题、发布插件、写使用指南；插件生态被明确鼓励，仓库自身被定位为「展示与灵感来源」而非唯一权威。
- **会话数据格式采用不可变世代 + 相邻迁移**：新版本可以读取并迁移旧会话，但**不承诺降级**——用新版本写过的会话，不应再拿旧版本读。
- **运行时要求 Node.js `^22.19.0 || >=24.0.0`**，并用 Corepack 管理的 pnpm 构建源码。
- **文档密度差异明显**：项目内部的工程文档与设计笔记非常丰富（面向维护者），但缺少一份面向外部读者的、自顶向下的连贯说明——这正是本 Wiki 试图补上的位置。

> 依据：`README.md:1-77`、`SAFETY.md:1-27`、`CONTRIBUTING.md:1-23`、`AGENTS.md:1-20`、`package.json:1-40`、`docs/session-format-status.md:1-55`、`packages/README.md:1-70`
