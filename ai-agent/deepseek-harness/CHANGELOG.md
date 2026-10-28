---
type: 变更记录
title: DeepSeek Harness（dsh）解读版次记录
description: 本知识库对 dsh 各次解读的基线版本、产出结构与结论变化汇总——0.2.1-rc.3 全量重做、相比 0.1.7-rc.2 的上游变化、校验记录与上游文档/代码不一致清单。
tags: [deepseek-harness, dsh, changelog, 解读版次]
---

# 变更记录

> 本文件记录**本知识库对 DeepSeek Harness（`dsh`）的解读版次**，以及每版相对于上一版解读范围与结论的变化。上游项目本身没有官方 changelog（发布以 git tag 为准），因此这里汇总的是"解读基线 → 解读要点"的对应关系，不是上游的逐条发布说明。

## [0.2.1-rc.3]（2026-10-27 落盘）

> 依据：`dsh-v0.2.1-rc.3` @ `1f81c047fc`（2026-10-23）。本次为**全量重做**：先删除此前基于 `dsh-v0.1.7-rc.2` 的整套解读，再按当前基线从零重新拆解关注面、重新选定章节结构、全部引用重新回源码定位。

**本轮产出结构（12 份文档）**

- `README.md`（总览：根因 / 概览 / 架构总览 / 一次对话链路 / 模块导读 / 亮点 / 风险 / 可借鉴 / 总结）
- `architecture-design.md`（叙述型架构设计说明，14 章 + 2 附录，零代码索引）
- 10 篇关注面子文档：`plugin-kernel` / `session-event-log` / `agent-loop-and-tools` / `memory-subagent-longrunning` / `llm-adapters` / `execution-world` / `external-capabilities` / `clients-and-distribution` / `data-model` / `deployment-and-operations`

**相比上一轮（基线 `0.1.7-rc.2`）在本轮解读中确认的上游变化**（按解读关注面归类，非穷尽）

- **组装与插件内核**：profile/bundle/patch 的层叠关系稳定为"空根 + 有序补丁"，`sdk-minimal` 仍是刻意不走 base 的例外；新增插件市场（索引格式 + 兼容性评估）与 Creator 模式的持久化安装路径；实验能力改为"以可选 bundle 形式随安装下发、默认关闭、在插件页显式勾选"。
- **会话与外设状态**：会话格式推进到 v4 并已被产品发布确立；相邻迁移实现改为**有状态流式 stage**（此前整文件迁移曾让 116 MB 会话耗尽 16 GB 内存）；`SessionPersistence` 收敛为 `create/open/flush/stat/list` 五方法，导出能力移到独立包。
- **主循环与工具**：系统提示词成为 surface node 0 的语义稳定；`cancelAll` 在当前基线**不存在**（统一为 `cancel(cause, { keepInbox })`）；工具结果补齐仍存在三套互不复用的实现（解读中列为已知不一致）。
- **执行世界**：E2B 系列 provider 退役，远端执行改由 SSH 系列（`fs-ssh` / `subprocess-ssh` / `sandbox-ssh`）成组承担；默认权限来自 base profile 的 `workspace-write` + `ask`（不是 schema 的 `read-only`）。
- **外部接入**：ACP 被明确为"dsh 作为 ACP 服务端被宿主拉起"（与 MCP 方向相反）；动态 Cordis 扩展的审批只覆盖 Client 半边，Host 半边靠上下文门面而非沙箱；MCP 保留"支持则保真、否则降级"的输出 schema 策略。
- **客户端与分发**：Typert 生成式双契约取代手写 RPC（严格模式只在产物模式下成立）；桌面改为"壳内回环 HTTP + 应用协议代理"，此前的无端口分帧管道被替换；桌面运行时为 Electron 自身当 Node + 主运行时载荷 + 构建期完整性清单，用户机器无需 Node/pnpm/Python。
- **数据与存储**：领域 KV 以"每域一份带版本戳 JSON"为主，`backup-and-skip` 只对 per-record 布局生效；SQLite 仅作为可选的会话查询索引存在，且 shipped profile 默认 `openAt: never`。
- **部署与远程访问**：远程访问收敛为"可达性 + Host/Origin 栅栏 + 进程令牌换签名 cookie"三层，"非 loopback 页面降级为 memory"的旧说法在源码中找不到对应；操作者身份与 Remote 方法级访问控制（principal + peer）落地。

**本轮校验记录**

- 源码引用：`verify-references.mjs` 对全部含引用文档重定位，结果为 `MISMATCH=0 UNVERIFIED=0 TRAVERSAL=0`（无锚点的引用降级为弱校验，列在 WEAK 中，均确认文件可读、行号未越界）。
- 图表：`check-mermaid.mjs` 对全部 mermaid 块做真实解析与渲染，14 张流程图 + 若干时序图全部通过。
- 叙述型文档：`check-narrative-all.mjs` 通过（零代码索引、编号唯一递增、交叉引用可解析、图非孤立插图）。

**本轮发现的上游文档与代码不一致（已按代码写入，不按文档）**

- wire 扩展文档称"每个提供方 HTTP 请求都带归属头（含 Files API）"，代码中 Files 请求未带。
- 桌面 README 称 Windows 安装目录可修改，实际配置为 `allowToChangeInstallationDirectory: false`。
- `storage-sqlite` 后端已实现但无任何 bundle 挂载，也无文档解释原因。
- 部分子系统参考页与代码在细节上有漂移（如迁移中间状态的描述），解读一律以源码为准。

**工具侧说明**

- 本轮校验中发现 `check-mermaid.mjs` 的一个环境相关缺陷：当图以 `graph` 关键字声明时会被解析为旧版 `flowchart` 渲染器，在纯 jsdom 下**任何带标签的边都会误报渲染错误**；改用 `flowchart` 关键字声明即可正常渲染（本轮已全部改用 `flowchart`）。
- 本轮校验中发现 `verify-references.mjs` 的三处覆盖盲区：① 已补真实链接（`[path:s-e](https://…)`）的引用不被识别；② `<!-- anchor-base64:首 … 末 -->` 双锚形式不被识别；③ 引用与指纹之间隔着说明行时，配对会错位到下一个指纹。本次通过工作区内的修改副本完成了校验，建议上游脚本合入。
