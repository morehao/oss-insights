---
type: 变更记录
title: OpenCode 解读变更记录
description: OpenCode 项目解读的跨版本变更汇总，按版本倒序，记录每个解读基线的依据 ref 与主要变更内容。
tags: [opencode, changelog, release-notes]
---

# 变更记录

> 本解读对 OpenCode 的跨版本变更汇总；按版本倒序。上游在 `CHANGELOG.md`、GitHub Releases 与 `V2_HTTP_API_AUDIT.md` 中维护发布与 API 变更，本文件只记录「解读内容」自身的基线变化与结论修订，不重复上游发布说明。

## [v2.0.11]

依据：`v2.0.11`@`9eb6902aaf`

本次为**全量重写**，删除并替换了此前的 5 篇旧文档（`README.md`、`architecture.md`、`llm-route-system.md`、`provider-config-guide.md`、`session-system.md`）。

重写的原因是可验证的，而非偏好：旧文档集**不含任何可程序校验的源码引用**（`verify-references.mjs` 对旧文的统计为 0 条 `文件:行号` 引用、0 条锚点指纹），且包结构已在上游发生实质变化——包作用域由 `@opencode-ai/*` 更名为 `@opencode/*`，`packages/llm` 已更名为 `packages/ai`（commit `bb15b16499`），单体包 `packages/opencode` 拆分为 `cli` / `tui` / `server` / `app`。旧文中十余处 `packages/llm/...` 路径在 v2.0.11 下全部失效。新文档集统一补齐 frontmatter、改为语义式标题，并为每条论断附上经程序重定位校验的真实 `文件:行号` 引用与逐字锚点指纹。

**基线说明：** 本次按 `v2.0.11` 解读，已确认接受其陈旧度。基线选择由 `select-baseline.mjs` 程序化判定并返回 `[BASELINE-STALE]`——`v2.0.11` 距 `dev` 分支落后 1261 个提交，且 v2 系列发布距今仅数日。选择该 tag 而非分支，是为了让全部引用锚定在一个不可变的快照上；引用只在 `v2.0.11` 工作树下有效。

文档集（`ai-agent/opencode/`）：

- `README.md` —— 总览：根因、项目定位与规模、技术栈、整体架构、子文档索引、核心模块导读表、同类定位、设计亮点、局限与风险、可借鉴的设计。
- `architecture.md` —— 总体架构与运行时骨架：依赖方向为何由可发布性而非 lint 强制、自建 `LayerNode` 图体系、`Global`/`Location` 两阶段部署、目录实例的生命周期与驱逐、进程模型与启动时序。
- `session-engine.md` —— 会话引擎：事件溯源账本、事实粒度、准入/投递/提升三阶段、ID 幂等、投递词表、执行权协调与 write-ahead claim、会话派生对事件历史与指令纪元的影响。
- `agent-loop.md` —— Agent 循环：step 与物理尝试的双层模型、四条恢复路径、显式 `llm.stream` 纪律、上下文装配与压缩的职责切分、指令代数与纪元、停止与失败语义、旁路调用。
- `tool-system.md` —— 工具系统：三类来源的统一注册与单一校验点、权限裁决模型、执行与错误传播、输出边界治理、文件变更影响面、内置工具深读。
- `llm-route-system.md` —— LLM 接入层：protocol / endpoint / auth / transport 四轴正交分解、复用度实测、Bedrock 反例、续写恢复、外部元数据目录、工具调用的线协议差异、错误归一化。
- `data-model.md` —— 数据模型：账本与投影视图的分界、提升列演进地层、ER 图与逐表字段表、Schema 演进与迁移门禁、v1→v2 一次性迁移、五个持久化域的数据边界、保留与清理。
- `mcp-and-plugins.md` —— 两条扩展路径：跨进程 MCP 与进程内插件，各自的连接生命周期、钩子注册时机与信任边界。
- `skill-system.md` —— Skill 与配置体系：分层发现与按需注入、指令纪元与增量更新、配置优先级与热重载、自定义命令。
- `acp-integration.md` —— 宿主接入协议 ACP：适配层的服务化真相（私有子进程 + 生成客户端）、工具执行全在 agent 侧、权限往返的取消语义、信息损失清单、明确未实现的部分。
- `server-and-protocol.md` —— 服务与契约：三层契约与生成管线、API 分组口径、中间件链与归属顺序、SSE 事件推送与溢出策略、两层授权、多运行时画像、OpenAPI 与 SDK 的自持方式。
- `code-mode.md` —— Code Mode：与逐个工具调用范式的对比、受限解释器与支持矩阵、默认全局表的逐项核对、工具到函数的签名塑造、失败分类学、与 MCP 的互斥机制、宿主中立边界。
- `tui.md` —— 终端前端：OpenTUI 渲染底座、应用骨架与单路由值、数据来源（HTTP + 事件流）、语义主题令牌、会话渲染的投影与增量路径、second frontend 定位、终端能力探测与降级。
- `web-and-desktop.md` —— 图形前端与周边：Web app 分层与会话 UI 边界、组件层与契约层的共享取舍、Electron 三进程模型、控制台与官网的定位区分。
- `platform-packages.md` —— 平台与支撑包：模拟环境（Drive）、HTTP 录制回放与 cassette 纪律、终端图表与公式渲染、自研持久层的造轮子取舍、浏览器 Code Mode 工具、服务与根级设施。
- `deployment.md` —— 部署与分发：安装器探测、分发形态矩阵（仓库有配置 vs README 宣传）、单二进制与资产内嵌、服务选举与注册文件、数据目录与 channel 隔离、自更新策略与无回滚、CI/CD 与发布链、自托管张力。

结论修订与新增发现（相对旧文档集）：

- 更正技术栈事实：Effect 为 `4.0.0-rc.112`（旧文档写作 `4.0.0-beta.83`）；Vercel AI SDK 在此并非「兜底方案」，而是被降格为 Endpoint / Auth / Framing 三轴退化为常量的**合成 route**。
- 更正架构事实：`packages/effect-drizzle-sqlite` 在 v2.0.11 已是空壳（无 `package.json`、无 `src/`），持久层集成已内联进 `packages/core/src/database/drizzle/`。
- 新增规格与代码的分歧清单：包括重试次数（规格四次 / 代码十次）、输出截断切法、指令来源清单、`loadInstructions` 函数不存在、metadata 体积测量无实现等。
- 新增「已实现但无消费者」清单：`ai/tool-runtime.ts` 的 `dispatch`、MCP 的 `readResource`、Code Mode 的 OpenAPI 适配器。
- 新增实测口径：HttpApi 操作为 136 个 / 113 条 path（与仓库内 API 审计清单所记的 139 存在漂移），并与 `groups/*.ts` 的计数双向互印。

本解读自身的覆盖局限已如实记录：`packages/merman` 仅覆盖约 6%，`packages/ai` 的协议实现与同构 provider facade 采用「关键区间精读 + 代表性抽样并证实同构」的口径，未逐行通读。
