# 自动化编排（Automations）

> 关注面子文档：OpenHands Agent Canvas 的自动化工坊——Automation Server 契约、manifest 安全收纳层、触发与 run 实时 phase。承接 README 主文档。

## 根因

自动化的本质是**让 agent 运行不再依赖人盯着页面触发**：定时（cron）或事件（webhook）触发一次 agent 运行。困难在于"触发的 agent 运行怎么被编排、怎么监控、怎么安全地由外部包扩展"。Agent Canvas 把自动化做成了**宿主 + 外部 manifest**的解耦：宿主只做准入校验与派生行为，模板由已发布 npm 包下发。

## Automation Server 契约：哑客户端 + 中间人

前端对 Automation Server 是**哑客户端**：base path `/api/automation`，REST 端点路径由外部 interface manifest 声明（`getAutomationEndpoint`），本地走 sidecar、云端走 cloud proxy，鉴权用 `X-Session-API-Key` `[src/api/automation-service/automation-service.api.ts:50,274-301,428-443]()`。端点路径不写死，从 interface manifest 解析 + `{id}` 编码 `[src/manifests/automation-interface.ts:133-143]()`。

两端路由：本地经 `localAutomationAxios` 拦截器每次解析当前活跃后端并注入 header `[src/api/automation-service/automation-service.api.ts:83-111]()`；云端经 `callCloudProxy`（避免 gzip body 被 JSON 序列化破坏，upload 直连 host）`[src/api/automation-service/automation-service.api.ts:634-660]()`。

## manifest 安全收纳层：外部包 → 宿主校验 → 派生行为

manifest 是**外部仓库（`@openhands/extensions`）声明的指令数据**，宿主不信任它，把"准入校验"当**信任边界**而非便捷检查——不 defer 给随包 schema，而是宿主自持校验规则。

- **来源**：catalog 作为 `unknown[]` 传入（故意不信同包类型）`[src/manifests/manifest-sources.ts:18-20]()`
- **收纳（registry）**：逐候选先 `hasSetupBlock` 再 `validateSetupEntry`，失败 `console.warn` 后**整体丢弃**（不渲染半成品），重复 id 也拒 `[src/manifests/manifest-registry.ts:25-46]()`
- **准入校验**：`setup.version` 必须精确等于宿主 `SETUP_VERSION`，否则 fail-closed（不认识就拒而非照今天规则乱解）`[src/manifests/manifest-validation.ts:576-592]()`
- **关闭集白名单**：命令只许安全字符且逐词禁绝对路径/`..`（防逃出解压目录）`[src/manifests/manifest-validation.ts:165-178]()`；字段名、trigger 单种、repo-picker 事件源、占位符只许已知 namespace、copy 禁 markup `[src/manifests/manifest-validation.ts:25-49]()`
- **派生行为**：请求体完全由主/表单推导（name、trigger、repos、prompt），条目只声明"差异"，镜像 `extensions` 侧 reference 实现 `[src/manifests/automation-setup.ts:185-347]()`
- **template provenance（本版新增）**：创建时把 `{id, version, config}` 作为 `template` 发送，服务端识别"同一条目再次 setup"复用已有 automation；prompt 条目取顶部 `version`，bundle 条目取 `setup.bundle.version` `[src/manifests/automation-setup.ts:236-246,314-318]()`

```mermaid
sequenceDiagram
    participant Ext as "extensions 包(catalog)"
    participant Reg as manifest-registry
    participant Val as manifest-validation
    participant U as UI/表单
    Ext-->>Reg: AUTOMATION_CATALOG unknown[]
    Reg->>Val: 逐候选校验 hasSetupBlock
    alt schema 校验失败 / 重复 id
        Val-->>Reg: console.warn → 整体丢弃
    else 通过
        Val-->>Reg: setup.version 精确匹配
        Reg-->>U: 干净条目
        U->>U: 表单推导请求体字段
        U-->>Server: 创建含 template={id,v}
    end
```

## 触发与运行状态机

### run ENUM 状态机 + 可实时更新的 phase

run 表面是 ENUM `PENDING/RUNNING/COMPLETED/FAILED/CANCELLED/SKIPPED`，叠加一层**可实时更新的 phase**（code+label+updated_at，服务端可逐步写入）`[src/types/automation.ts:66-73,99-113]()`。

**"该不该展示 phase"派生**：只有 `PENDING/RUNNING/FAILED` 值得展示（完成/取消/跳过没内容）`[src/components/features/automations/detail/run-phase.tsx:17-25]()`。**"该不该展示 age"**：只有仍在途的才展示 age，age 唯一作用是分离"在跑 vs 卡住"；失败态显示 age 会退化成第二时间戳 `[src/components/features/automations/detail/run-phase.tsx:35-42]()`。

**phase→文本唯一入口**：已知 code 走翻译表（queued/sandbox_provisioning/...），未知 code 用作者 label、再兜底 raw code；用 `Object.hasOwn` 防 `toString` 原型污染 `[src/components/features/automations/detail/run-phase.tsx:65-102]()`。

```mermaid
stateDiagram-v2
    [*] --> PENDING
    PENDING --> RUNNING: dispatch 接受
    RUNNING --> RUNNING: phase_code 实时更新
    RUNNING --> COMPLETED
    RUNNING --> FAILED
    RUNNING --> CANCELLED: 用户 cancel
    PENDING --> SKIPPED
    FAILED --> COMPLETED: (可恢复?)
    COMPLETED --> [*]
```

### 健康度派生

`checkHealth()` 5s 超时 fail-fast `[src/api/automation-service/automation-service.api.ts:762-787]()`；`useAutomationHealth` 30s stale、失败不重试（立即展示错误态）`[src/hooks/query/use-automation-health.ts:7-14]()`；`automations-list` 先 gate health 再拉 automation 列表——`isBackendHealthy` 为 false 就跳 `BackendNotConfigured` 而非拉列表 `[src/routes/automations-list.tsx:131-133,350-358]()`。

## 全局关联与权衡

自动化的编排哲学是"**宿主只编排、模板只声明差异、服务端保持续**"：请求体完全由当前表单推导，manifest 条目只声明与基准的差异 + template provenance 指纹（id/version），使服务端能识别"同一模板再次安装"并复用，避免重复创建。这与控制面"隔离不确定性"同源：前端不解析执行细节，只做准入校验、派生行为和 run phase 呈现；真正执行是 Automation Server + agent-server 的事。代价是对已发布 manifest 结构高度耦合，包升级可能破坏承载 UI。
