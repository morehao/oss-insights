# tutti 项目深度洞察报告

> 分析时间: 2026-07-09
> 项目: [tutti](https://github.com/anomalyco/tutti) — "Where people and agents build in tune."

---

## 1. 项目概览

tutti 是一个**本地优先（local-first）的桌面端多智能体协作平台**。核心理念是让多个 AI 智能体（Claude Code、Codex、OpenClaw、Hermes 等）在同一实时共享工作空间中协作，无需手动复制粘贴上下文。提供应用中心、任务编排、实时共享 workspace 等功能。

### 关键数据

| 维度 | 数值 |
|---|---|
| 技术栈 | TypeScript (前端) + Go (后端守护进程) |
| 包管理器 | pnpm 10.11.0 + Go workspace |
| 总源文件数 | ~2,700+ |
| 总代码行数 | ~300,000+ |
| API 端点 | 112 个 (OpenAPI 3.0.3) |
| Monorepo 包数 | 27 个 npm 包 + 多个 Go 包 |
| 测试/生产代码比 | ~52% (Go), ~37% (TS) |

---

## 2. 架构亮点

### 2.1 严格的分层架构

```
┌────────────────────────────────────────────────────────────┐
│                     apps/desktop (Electron)                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌───────┐  │
│  │ main/    │  │ preload/ │  │ renderer/    │  │shared/│  │
│  │ (Node.js)│  │(桥接层)  │  │ (React UI)   │  │(契约) │  │
│  └──────────┘  └──────────┘  └──────────────┘  └───────┘  │
└──────────────────────┬─────────────────────────────────────┘
                       │ HTTP/WS (127.0.0.1:4545)
┌──────────────────────▼─────────────────────────────────────┐
│                  services/tuttid (Go 守护进程)               │
│  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌────────────┐  │
│  │ api/    │→ │ service/ │→ │ biz/    │→ │ data/      │  │
│  │(传输层) │  │(用例编排)│  │(领域模型)│  │(持久化)    │  │
│  └─────────┘  └──────────┘  └─────────┘  └────────────┘  │
│                    ↕ 事件发布/订阅                           │
│              ┌────────────────────────┐                    │
│              │  eventstream/ (消息总线)│                    │
│              └────────────────────────┘                    │
└────────────────────────────────────────────────────────────┘
```

核心原则：
- **守护进程是唯一业务核心**，Electron 壳层禁止包含业务逻辑
- **不创建模糊包**：禁止 `shared`、`common`、`utils`、`client-sdk` 等无责任边界的包名
- **ORMs / DI 框架零依赖**：手写 `wiring.go` 做依赖组合，Go 标准库 + SQLite 驱动

### 2.2 包依赖 DAG (有向无环图)

```
Layer 0: config-tsconfig, config-prettier
Layer 1: event-protocol, event-stream-core, app-release-tools
Layer 2: ui-system, ui-i18n-runtime, ui-react-hooks, ui-rich-text
Layer 3: workbench-snapshot
Layer 4: workbench-surface, workspace-* (TS 包)
Layer 5: agent-gui, agent-activity-core
Layer 6: browser-node
```

所有边方向一致，**无循环依赖** —— 这是大型 monorepo 中最难维持的纪律之一。

### 2.3 Schema-first 事件协议

events 包实现了真正的 schema-first 跨语言事件协议：

```
definitions/agent/agent-context.ts         # schema 定义
definitions/workspace/workspace-context.ts
     ↓ (自动生成)
src/generated/contracts.ts                  # TypeScript 合约
src/generated/validators.ts                 # 运行时验证器
src/generated/registry.ts                   # 事件注册映射
     ↓
packages/events/stream-go/contract.go       # Go 端对应合约
```

Go 端和 TypeScript 端保持一致的合约定义，确保跨语言通信的类型安全。

---

## 3. 代码质量分析

### 3.1 强项

| 维度 | 等级 | 说明 |
|------|------|------|
| **架构清晰度** | A | 4 层架构 (desktop) + 4 层架构 (tuttid)，职责分明 |
| **类型安全** | A | 全量 TypeScript + 全量 Go，IPC 通道有泛型类型约束 |
| **安全性** | A- | contextBridge 隔离、类型安全 IPC、entitlements 配置 |
| **i18n 国际化** | A | 英文/中文双语，所有用户可见文本强制走 i18n 层 |
| **测试覆盖** | B+ | Go 52%，TS 37%，含集成测试 |
| **代码规范** | A | Oxlint + golangci-lint + 10+ 自定义边界检查 |
| **文档文化** | A | 每次变更后需评估文档影响，自进化文档体系 |

### 3.2 技术债务 (按优先级)

#### P0 — AGENTS.md 硬规则违规

以下文件违反"业务代码不超过 800 行"规则：

| 文件 | 行数 | 所属模块 |
|------|------|----------|
| `WorkspaceSettingsPanel.tsx` | **4,701** | desktop/renderer |
| `service/agent/service.go` | 828 | tuttid/agent |
| `service/agent/external_import.go` | 820 | tuttid/agent |
| `service/agent/activity_projection.go` | 842 | tuttid/agent |
| `service/workspace/app_factory.go` | 858 | tuttid/workspace |
| `service/workspace/apps_runner.go` | 840 | tuttid/workspace |
| `service/agentstatus/service_helpers.go` | 829 | tuttid/agentstatus |
| `service/managedruntime/runtime.go` | 814 | tuttid/managedruntime |
| `api/routes.go` | 886 | tuttid/api |
| `workspaceAgentActivityService.ts` | 2,100 | desktop/renderer |
| `desktopAgentProviderStatusService.ts` | 1,275 | desktop/renderer |
| `desktopRichTextAtService.ts` | 1,251 | desktop/renderer |
| `workspaceAppCenterService.ts` | 1,423 | desktop/renderer |
| `workspaceSettingsService.ts` | 1,375 | desktop/renderer |
| `workspaceWorkbenchHostService.ts` | 1,379 | desktop/renderer |
| 其他 renderer 文件 10+ 个 | 800-1,500 | desktop/renderer |

**总计约 30+ 个文件违反 800 行限制**。最严重的是 `WorkspaceSettingsPanel.tsx` (4,701 行)。

#### P1 — 测试覆盖缺口

| 薄弱环节 | 现状 | 风险 |
|---------|------|------|
| `apierrors/` | **0 个测试文件** | 错误映射是跨层协议关键，无测试覆盖 |
| preload 层 | 仅 5 个测试文件 | 安全边界层测试不足 |
| `api/` HTTP handler | 30% 覆盖 | handler 逻辑缺少集成测试 |
| `biz/` 领域层 | 14% 覆盖 | 领域逻辑随增长需更多保障 |

#### P2 — 架构体感问题

1. **`agent/gui` 包边界过宽** — 暴露 23 个 subpath exports，实际承担了多个逻辑域（消息中心、AgentComposer、工作台、@面板），建议考虑拆分
2. **`analytics/` 碎片化** — 263 个文件仅 3,034 行，每个事件一个文件，过于碎片
3. **`buildDaemonAPI()` 函数过长** — 256 行手工 DI，建议拆分为子模块构造函数
4. **Store 类型转型不安全** — `wiring.go` 中 `store.(workspacedata.XxxStore)` 若接口未实现会静默返回 nil

#### P3 — 可优化项

1. 缺少 CSP (Content Security Policy) 配置
2. 无端到端 Electron 集成测试
3. `src/main/` 根层 41 个文件密度偏高
4. `agent/gui` 的 `agentGuiNode/` 目录下 53 个文件平铺在同一级

---

## 4. 工程实践亮点

### 4.1 多层次验证体系

```
pnpm check:changed       # 日常快速验证
pnpm check:full          # CI 全量验证 (pre-push hook)
pnpm lint:ts             # Oxlint TS 检查
pnpm typecheck           # tsgo 增量类型检查
pnpm check:ui-boundaries # UI 包边界检查
pnpm check:renderer-boundaries # 渲染器特征边界
pnpm check:i18n          # i18n 完整性检查
pnpm check:api-generated # API 生成一致性检查
pnpm lint:go             # golangci-lint Go 检查
```

**关键设计**：`check:changed` 只运行改动影响的检查，并行执行，存储全量日志到 `.tmp/check-runs/`，通过 `--tail-lines` 控制失败输出长度。这是大型 monorepo 中提升迭代速度的典型做法。

### 4.2 文档自进化 (AutoSkill)

团队通过 AutoSkill 四态决策（discard/improve/merge/create）维护文档：

> "每次代码变更后，如果影响模块职责、数据流、用户交互、公共 API/CLI 行为、运行时/配置/环境覆盖、验证命令、排错路径或目录职责，需更新对应的持久文档。"

这是一个值得借鉴的做法——将文档维护嵌入开发流程而非事后补救。

### 4.3 Git Hooks 设计

```
pre-commit: lint-staged (仅检查暂存文件)
pre-push:   pnpm check:full (全量检查)
```

**设计思路**：pre-commit 快速反馈（仅改动的文件），pre-push 质量门禁（全量检查）。避免等待 CI 就发现问题，也不让 pre-commit 耗时过长。

---

## 5. 架构决策分析

### 5.1 Local-First 架构

全部状态存储在本地 SQLite，路径约定：
- 开发: `~/.tutti-dev/tuttid.db`
- 生产: `~/.tutti/tuttid.db`

**优势**：离线可用、低延迟、用户数据主权。
**隐含成本**：多设备同步、数据备份、冲突解决等方案需自行实现。

### 5.2 无 DI 框架

手写 `wiring.go` 管理 25+ 组件的依赖注入，AGENTS.md 明确禁止第三方 DI 框架。

**合理性**：小型项目手写 DI 的维护成本低于学习框架成本。但当前 `buildDaemonAPI()` 已 256 行，若继续增长应考虑拆分子函数（`buildAgentModule`、`buildWorkspaceModule` 等），而非引入 DI 框架。

### 5.3 OpenAPI Contract-First

API 变更流程：
1. 修改 `services/tuttid/api/openapi/tuttid.v1.yaml`
2. 通过 `pnpm generate:api` 生成 Go handler 接口 + TypeScript 客户端
3. 通过 `pnpm check:api-generated` 确保一致性

使用 fragments 机制 (`x-tutti-openapi-fragments`) 将 `issue-manager.v1.yaml` 等片段组织到主规范，避免单一巨型文件。

### 5.4 跨语言契约

| 契约层面 | TypeScript 实现 | Go 实现 | 同步机制 |
|----------|----------------|---------|----------|
| HTTP API | `@tutti-os/client-tuttid-ts` | `api/generated/` | OpenAPI → oapi-codegen |
| 事件协议 | `event-stream-core` | `event-stream-go` | schema-first 手工映射 |
| Workbench | `workbench-surface` | `workbench/service` | 共享接口定义 |

---

## 6. 项目风险与建议

### ⚠️ 主要风险

1. **Service 层膨胀**：`service/agent/` 已 76 个文件，是最大的复杂度来源。随着更多 agent 类型（Claude Code、Codex、Hermes 等）的加入，这一模块的边界管理将面临挑战。
2. **Store 接口的安全转型**：`wiring.go` 中未检查的 `store.(workspacedata.XxxStore)` 转型在运行时可能 `panic`，虽然 Go 的接口检查不是大问题，但在启动阶段静默失败会增加调试难度。
3. **agent/gui 包边界**：23 个 exports 暗示该包正在成为一个"上帝包"，建议细化为更小粒度的包。

### 🎯 推荐行动项

| 优先级 | 行动 | 预期效果 |
|--------|------|----------|
| P0 | 分解 30+ 个超 800 行文件 | 恢复 AGENTS.md 合规，提升可维护性 |
| P0 | 为 `apierrors/` 增加测试 | 消除严重测试缺口 |
| P1 | 增加 preload 层测试覆盖 | 加固安全边界 |
| P1 | 拆分 `agent/gui` exports | 明确包边界，避免上帝包 |
| P2 | 重构 `buildDaemonAPI()` 为子模块 | 提升 DI 代码可读性 |
| P2 | 添加 CSP 配置 | 提升 Electron 安全性 |

---

## 7. 对开源社区的启发

tutti 在以下方面展现了值得学习的工程实践：

1. **Monorepo 治理**：pnpm workspace + Go workspace 共存，依赖 DAG 无环，边界检查自动化
2. **Local-First 桌面应用**：Go 守护进程 + Electron UI 的 hybrid 架构，提供低延迟本地体验
3. **文档自进化**：AutoSkill 模型将文档维护编码为开发流程的一部分
4. **多 Agent 编排**：支持多种 AI 智能体在同一 workspace 协作的设计模式
5. **分层验证**：从 pre-commit 到 pre-push 的验证链，兼顾速度和安全性

---

*本报告基于代码静态分析和目录结构推导，部分数据（如代码行数）为近似值。*
