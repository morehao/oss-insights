# 变更记录

> 本解读对 Lework（Leros，`github.com/insmtx/Leros`）的跨版本变更汇总；按版本倒序。描述清楚相关变动即可：上游有官方 CHANGELOG（`CHANGELOG.md`），此处参照它并结合代码变动生成。

## [v0.5.2] - 2026-08-25

> 依据上游 CHANGELOG v0.5.2 + git diff / 代码快照比对生成。

- **新增「工作台/考勤工资核算」场景**：前端侧边栏新增工作台视图，演化出 `PayrollWorkbench` 考勤+工资核算页；后端把核算技能收敛为**场景内置 Skill**（`salary_accounting` → 强制安装 `attendance-payroll`）。
- **AI 队友共享权限**：新增 `Get/UpdateDigitalAssistantPermissions` 读写成套，AI 队友可被组织内多成员以角色共享治理；数据层复用 `ResourceBinding`，不新增表。
- **产物/任务文件时间按日历日展示**：前端 `formatArtifactTime` 按本地日历日区分「今天/昨天/具体日期」。
- **脱敏客户信息 + 按文件名匹配用工类别**：考勤工资核算的敏感字段脱敏与匹配收敛。
- 结构层：新增 `automation.md`、`model-router.md` 两篇关注面子文档；`data-model.md` 补 `automation` / `automation_execution` 两表；`mcp-plugin.md` 补配置驱动系统连接器与共享权限；frontend / service-layer / agent-runtime / worker-system / skill-tool 就地补充对应增量。

## [v0.4.1] - 2026-08-20

> 依据上游 CHANGELOG v0.4.1 + git diff / 代码快照比对生成。

- **Worker Skill 版本缓存**：新增 `.seed-manifest` 安装清单，`revision + sha256` 未变则跳过重复下载/解包。
- **任务 Skill 目录清理**：run 结束时 `cleanup` 闭包 import 回写任务 Skill 目录 → 恢复 worker 仓库 → 删除 run 视图。
- **技能项目关联 / 隐藏 Worker 内置技能 / 技能引用与本地目录说明**：技能来源可见性与候选标签完善。
- **Kubernetes 按 Deployment 隔离 workspace**：`workspacePath(orgID, workerID)` + HostPath/MountPath 隔离 + `workspaceSpecDrifted` 漂移检测。
- **统一对外输出边界**：前端 HTML 源码预览（预览/源码双视图）+ 统一 Markdown 代码块主题。
- 私有化部署资源 / 模型 / 并发权衡文档补充（非产物变更）。

## [v0.4.0] - 2026-08-14

> 依据上游 CHANGELOG v0.4.0 + git diff / 代码快照比对生成。

- **标书对比场景**：项目页/任务续聊发起对比，强化对照产出。
- **默认 LLM 模型**：支持设置默认模型；克隆系统模型时保留 `base_url` 是否含 `/v1` 标识。
- **系统 MCP 连接器凭据统一绑定**：系统连接器模板凭据绑定收敛；CatAPI 通过配置驱动接入（Bearer 绑定）。
- **多人协作项目不再向任务注入连接器**：`shouldDisableProjectMCP` 在 `humanCount >= 2` 时剔除 MCP 连接器注入。
- **OpenCode 固定配置目录 + 隔离任务级 Skill**：`OPENCODE_CONFIG_DIR` 固定 + 多环境隔离变量 + per-run Skill 注入。
- **Worker 镜像默认 IfNotPresent 拉取**；移除 `CallerHTTP` 内部超时。
- **聊天代码块亮暗主题切换并记住偏好**（前端）。

## [v0.3.12] - 2026-08-12

> 依据上游 CHANGELOG v0.3.12 + git diff / 代码快照比对生成。

- **MCP 连接器按配置初始化**：系统连接器从内置 seed 改为「Server 配置声明 → `SyncConfiguredMCPConnectors` reconcile → 幂等落库」，支持批量连接器配置。
- **AI 队友身份约束**：worker 侧 system prompt 追加 `<identity_constraints>`，自我介绍输出稳定。
- Helm 支持按配置初始化系统 MCP 连接器。

## [v0.3.11] - 2026-08-12

> 依据上游 CHANGELOG v0.3.11 + git diff / 代码快照比对生成。

- **系统级模型管理**：新增模型管理（组织创建者、必填校验、仅停用可编辑、连接测试、启用/禁用）。
- **独立组织管理页**：纳入模型管理入口。
- **自动化调度增强**：指令中选技能、多选执行周期、关联已有项目；修复创建停用状态被覆盖。
- **Eino 迭代上限 50**：`pkg/eino/flow.go` 的 `DefaultMaxIterations = 50`。
- **Worker 基础镜像装 xlrd**（读 `.xls`）。
- **私有化部署 UI 抽到 app-ui**：apps 仅剩 web/desktop 两个入口壳。
- OpenCode native 透传采样参数与 budget input。
- ⚠️ CHANGELOG 声称「worker CLI 默认值改为 opencode」，但当前源码 `cli.default` 仍为 claude、flag 空串兜底 Leros——存在版本差异，已在解读中按源码如实标注而非照搬 CHANGELOG 措辞。（此差异随版本演进可能再调整。）

## [v0.3.10] - 2026-08-10

> 依据上游 CHANGELOG v0.3.10 + git diff / 代码快照比对生成。

- **助手回复展示 Token 消耗**：前端 `MessageTokenUsage` 指示器（输入/输出用量）。
- **OpenCode 按模型注入采样参数**：`top_p/frequency_penalty/presence_penalty` snake_case 键透传。
- Eino ReAct 迭代上限调整为 30（后于 v0.3.11 再抬至 50）。

## [v0.3.9] - 2026-08-07（纳入增量范围起点）

> 依据上游 CHANGELOG v0.3.9 + git diff / 代码快照比对生成。此为既有解读基准（约 8/04）之后的第一个版本。

- **Agent 定时自动化**：新增 Automation 子系统（`automation` 配置 + `automation_execution` 执行记录、Planner/Dispatcher 两阶段调度、遗漏折叠、CAS 防重、租约领取）——催生本次新增 `automation.md`。
- **OpenCode 图片多模态附件**：视觉能力按模型声明优雅降级、多模态收窄为仅图片。
- worker-base 拆分 saas/private 两版次。
