# 变更记录

> 本解读对 TripStar 的跨版本变更汇总；按版本倒序。上游未维护官方 changelog，故以本地仓库的 git 提交与代码快照比对为依据。

## [首次解读] main@1dce6df

依据：main@1dce6df（全新模式首次解读，无上一版本可对比）

本条目建立解读基线。TripStar 上游仓库无 tag、无官方 changelog，`select-baseline.mjs` 按「稳定 tag > 稳定分支 > 默认分支」矩阵回退到默认分支 `main`，工作树干净、无需检出。本次为**全新解读**，产物为项目 `README.md` 与 8 篇关注面子文档 + 本文件。

### 覆盖范围（新增能力）

- `README.md`：项目定位、根因、四层架构总览、核心能力清单、ai-agent 规格覆盖表、项目规模、同类定位对比、模块导读、上手路径、设计取舍与不足。
- `multi-agent-orchestration.md`：3 个 `SimpleAgent`（天气/酒店/规划）分工、私有文本协议工具调用 `[TOOL_CALL:name:key=val]`、每城市串行主循环与进度切片、LLM 降级路径、五层 JSON 容错修复链。
- `memory-system.md`：带权重偏好记忆的准入阈值 / 同文本合并 / 惰性时间衰减 / TOP-K 召回 / 长度截断四道闸门，以及内存与 SQLite 两种存储的语义差异。
- `xhs-integration.md`：PyExecJS 原生签名引擎与 300011 风控绕过、笔记详情 SSR 降级、LLM 提纯为三语姓名三元组与预约提醒、高德地理编码限流并发、图片防盗链代理与两级缓存 + SSRF 白名单。
- `map-provider.md`：Google 优先 / 高德兜底的统一调度、中英文地址按供应商路由、地理编码失败进程级短路与显式重置、MCP 通路与 REST 兜底通路的分离。
- `async-task-system.md`：进程内任务表 + 磁盘原子快照、重启后未完成任务标记失败、`asyncio.Queue` 广播与死订阅清理、WebSocket 快照优先协议、轮询兼容接口。
- `knowledge-graph.md`：`TripPlan` 到 ECharts 力导向图数据的纯函数变换、样式与 i18n 分层、多城市合成根节点、景点顺序链、天气节点按日期去重的缺陷。
- `frontend-rendering.md`：同源 API 基址与长任务零超时、WebSocket Promise 封装、`sessionStorage` 快照与后端回补、按 tab 懒初始化、地图初始化代次校验与 5 秒超时降级、图片并发池与 name 键代理。
- `data-model.md`：`TripPlan` 聚合的实体关系、Pydantic 契约对 LLM 输出的容错、任务快照与 `user_memory` 两类持久化形态、读写路径、schema 演进策略与卷挂载边界。

### 解读中确认的现状与缺口

以下均为**首次解读的现状描述**（非版本变更），记录以便后续增量核对：

- **上游 README 与实际实现不一致**：上游宣称的并发执行阶段在代码中不存在（全仓无 `asyncio.gather`），实际为按城市顺序串行推进。
- **`AmapService` 为半成品桩实现**：`search_poi` / `get_weather` / `plan_route` / `geocode` 调用 MCP 工具后统一返回空值并标 `TODO`，真正生效的高德能力在 Agent 侧与 REST 兜底路径上。
- **前端 `resumeTripPlan` 为死代码**：已实现断线重连逻辑但无任何调用点。
- **记忆双存储语义不一致**：内存存储召回时会写回衰减与访问时间（等效续期），SQLite 存储不写回（读取不续期），行为差异静默无告警。
- **持久化卷边界不完整**：运行时设置文件与 SQLite 记忆库默认路径均在 `backend/data` 卷之外，容器重建即丢失，且无提示。
- **单实例硬约束**：任务表为进程内字典且启动脚本固定 `--workers 1`，无法水平扩展与滚动升级；任务无上限、无并发控制、无取消入口。
- **多城市天气节点冲突**：天气节点 id 仅用日期，多城市行程同日期天气会被去重丢弃。
