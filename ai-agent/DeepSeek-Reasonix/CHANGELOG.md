---
type: 变更记录
title: DeepSeek-Reasonix 解读变更记录
description: 面向检索：DeepSeek-Reasonix 项目解读的跨版本变更汇总（按版本倒序），记录每个解读基线的依据 ref、覆盖范围、校验口径与结论修订。
tags: [reasonix, changelog, release-notes]
---

# 变更记录

> 本解读对 DeepSeek-Reasonix 的跨版本变更汇总；按版本倒序。本篇只记 Reasonix 自身，上游依赖与同类项目的解读见同类目录下的独立解读。

## [首次解读]（2026-10-12 落盘）

依据：`v2.37.0`@`e17ef45e3`

首次解读，无"上一版"可对比，因此本条记录的是**本次解读的覆盖范围与基线状态**，而非相比上一版的变更：

- **基线选择**：仓库共 561 个 tag，其中 44 个属于 `studio-v2.x.y` 稳定线，另 517 个属于桌面（`desktop-v0.x`）等其它版本线。默认分支为 `studio`，而 `README.md` 的 Versions 表规定 2.x 从 `studio` 分支发布、1.x 从 `main-v2` 维护。按"稳定 tag > 稳定分支 > 默认分支"的次序取最新稳定 tag `studio-v2.37.0`（= `v2.37.0`，commit `e17ef45e3`，2026-10-11）。该 tag 落后 `studio` 分支 4 个 commit；解读完成后工作树已恢复到原分支 `studio`（`59b2b75e6`）。
- **产物**：`README.md`（总览、模块导读、同领域定位对比与分析章节）、`architecture-design.md`（叙述型架构设计，14 章 + 2 附录）、13 篇证据型子文档（Agent 回合循环、上下文与记忆、工具与沙箱、扩展体系、ACP 宿主协议、会话与检查点状态、数据模型、架构门禁、子 Agent 委派、模型 Provider 层、本机服务端与 Studio 通道、平台能力面、交付与部署）、16 张分层/关系 SVG，以及本文件。
- **引用校验**：全部 14 篇带引用的文档共 **2650 条** `[文件:行号]()` 引用经程序双向锚点校验通过（OK=2650 / WEAK=0 / MISMATCH=0 / UNVERIFIED=0 / TRAVERSAL=0）；每条引用下方由脚本烙入一行 `<!-- anchor: … -->` 指纹，供后续增量解读在源码漂移后重跑校验。16 张 SVG 全部通过端点拓扑审计并各带 `<title>`/`<desc>`；叙述型文档通过 `scan-narrative` / `check-mermaid` / `audit-svg-topology` 三项检查（error=0 / warn=0）。
- **覆盖到的自有能力**：回合边界与缓存稳定的前缀纪律（两级 turn tail 装配、投影级 CAS 折叠、per-turn 状态纪律）、指令与记忆的双通路切分（规则进前缀 / 事实出前缀）、叶子处权限裁决与三维正交沙箱、四层扩展面（MCP / Extension Protocol v2 / 插件包 v2 / Skill）、ACP 与 MCP 两套宿主协议、检查点与 rewind 的两段式事务与覆盖率缺口类型化、五个存储根与两个 SQLite 投影库、28 条自研架构门禁及其棘轮计量、子 Agent 的五概念边界与写声明调度、Provider 契约与两级重试预算归属、本机 HTTP/SSE 服务端的六道门与多工作区、远端工作区与自更新信任链。
- **明确标注为未交付/未达标的部分**：GA gate G3（会话与配置格式版本化并在 2.x 线冻结）仍为 Open，G1/G4/G5 同为 Open；发布列车规则 R6/R7 明确标注 "Planned, not in place"；Windows 上沙箱三个维度全部不强制；检查点保留策略为硬编码常量、无配置面；发布流水线不运行仓库规范门禁；`internal/platform/computer` 的 `CodeUnsupported` sentinel 在全仓无构造点。本次解读各关注面的次要文件覆盖率不足处已在各子文档中如实记录，未做虚报。
- **解读过程中确认的文档/实现不一致**（已写入对应子文档，供后续增量复核）：项目记忆文件 `REASONIX.md` 引用了已不存在的标识符 `agent.perTurnState`，其 `control` 准入名单漏列 TUI 与 benchmarks；`docs/SPEC.md` 的 §2 Layout 列出的全部 11 个 `internal/` 包路径均已搬迁，且其引用的 §Remote 小节不存在；`docs/CHECKPOINTS.md` 与实现存在代差、其保留期描述无代码依据；`docs/CONFIG_PATHS.md` 与 `docs/USAGE_CATALOG.md` 引用了不存在的 SQLite catalog 路径与错误版本号；`docs/SESSION_MEMORY_RETRIEVAL.md` 关于指令去向与 pinned 上限的两处描述与代码相反；`docs/RECOVERY.md` 描述的一个更新处理按钮在代码中不存在且未覆盖会话崩溃恢复；顶级 `README.md` 的扩展协议版本号、Go 版本要求、预发布候选描述与安装产物清单均与代码/发布作业不符；`.goreleaser.yaml` 在全仓无调用者且其引用的 npm 发布工作流不存在。以上均以源码口径为准记录。
- **本次未做的事**：本清单中的 2650 条引用均为**引用的可核性**校验（文件存在、行号在界内、首末行指纹双向匹配），**不等于每条论断的语义判断都被程序证实**；论断本身仍需人工复核。未提供性能与容量实测基线（叙述型文档第十二章走"可校准口径"）；未做端到端渗透验证与 Windows 实机验证；未运行 `go test`、`go run ./tools/repolint` 与发布流水线（源码门禁的债务数字取自 `tools/repolint/baseline.json` 的解析结果，而非实时运行）；未修改源码仓库中的任何文件。

后续每次增量解读时，在本文件顶部新增条目，首行记录上一条目基线到本次基线的依据（`依据：<旧 ref>@<短SHA> → <新 ref>@<短SHA>`）与上游 diff 或快照比对的说明；历史条目原样保留。
