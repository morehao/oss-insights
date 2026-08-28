# 变更记录

> 本解读对 OpenHands Agent Canvas 的跨版本变更汇总；按版本倒序。描述清楚相关变动即可：若上游有官方 changelog，参照它并结合代码变动生成；没有则基于代码变动（git diff / release notes / 代码快照比对）生成。

## v1.17.0（main@`226a6d2e6`，2026-08-28）

> 依据：上游 git log（`v1.16.0..HEAD`，共 2 个提交）。本地未打 v1.17.0 tag，以 main HEAD `226a6d2e6` 为锚。

- 新增能力
  - **manifest 声明的 value statements 展示**：dashboard 卡片与行渲染 manifest 声明的指标/值（`#16921`），指标实现集中在 `src/manifests/automation-insights.ts`
- 交互与接入变化
  - **会话 ACP 模型重派生**：chat reload 时从事件历史重新派生 active profile stamp，避免刷新后模型 chip 与运行态不一致（`#16439`）

## v1.16.0（commit `64c126965`，2026-08-27）

> 依据：上游 git log（`150e76046..v1.16.0`）+ 上游 release notes。相对解读基线 `150e76046`（v1.15.0 后）的主要变更如下。

- 新增能力
  - **Canvas Extensions 扩展前端**（`#16895`）：扩展管理页 + 页面贡献加载 + sidebar/customize；第三方以 `canvas-extension.json` 声明页面，运行时由 host 提供 `registerPage` 与 `agentServer.request` 代理
  - **pin favorite sidebar page as home route**（`#16868`）：`/` 重定向到用户固定的侧栏页，pin 按 backend+org 作用域持久化
  - **remember local workspace mode**（`#16881`）：上次本地 workspace 选择（`local_repo`/`new_worktree`）跨会话记忆
  - **skip onboarding when usable LLM**（`#16931`）：用户自建 Local 后端已有可用 LLM 时跳过 onboarding
- 交互与接入变化
  - **automation run-history fan-out 限流**（`#16867`）：每 automation 一个 runs 请求的 fan-out 压进 `ConcurrencyLimiter(3)`，轮询间隔放宽到 15s、失败不重试、失焦不刷新
  - **API key 前置校验**（`#16840`）：后端连接 step 在推进前先验证 key
  - **cloud 会话标题重命名**（`#16966`）：cloud 后端上会话标题可重命名
  - **default/free models 修正**（`#16922`）：默认与免费模型列表修复
  - **移除 unused random tip 代码**（`#16935`）
- 接口与 Schema 演进
  - **SDK 依赖升级**（`#16968`）：software-agent-sdk 1.44.0 / automation 1.9.0 / extensions 0.19.0
  - **`AutomationRunsResponse.status_counts`**（automation 1.9.0）：lifetime 计数按状态稀疏上报，不受分页影响；旧服务缺省时降级为样本计数或 `null`
  - **`Automation.preset_metadata`**：service-owned preset 状态原样回传，GUI 只读其中 template provenance 块且须经 guarded helper

## v1.15.0（commit `150e76046`，2026-08-25）

> 依据：上游 release notes + 代码快照比对。本版为解读基线（既有 v1.15 全量重写稿），此处仅登记版本锚点，详细解读见 README 主体。
