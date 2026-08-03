# OpenWorker 权限与安全模型深度分析

## 1. 权限引擎（PermissionEngine）

**文件:** `coworker/permissions.py`

PermissionEngine 是 OpenWorker 安全体系的核心——它不是简单的"允许/拒绝"门控，而是一个**基于风险等级的决策引擎**。

### 1.1 风险分类

**文件:** `coworker/risk.py`

每个工具调用在执行前都被分配到以下风险等级之一：

| 等级 | 枚举值 | 示例工具 | 默认行为 |
|------|--------|---------|---------|
| **读** | `RiskLevel.READ` | 文件读、搜索、网页抓取 | 自动允许 |
| **写** | `RiskLevel.WRITE` | 文件写、Git 操作 | 需要审批 |
| **Shell** | `RiskLevel.SHELL` | Shell 命令执行 | 需要审批 |
| **发送** | `RiskLevel.SEND` | Slack 消息发送 | 需要审批 |
| **MCP** | `RiskLevel.MCP_*` | MCP 工具调用 | 按工具配置 |

```python
@dataclass(frozen=True)
class RiskLevel(Enum):
    READ = "read"         # 无风险，自动允许
    WRITE = "write"       # 需要审批
    SHELL = "shell"       # 需要审批
    SEND = "send"         # 需要审批
```

### 1.2 权限模式

用户可以在对话中选择四种权限模式：

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| **Discuss** | 只读讨论模式，无编辑压力 | 需求讨论、代码审查 |
| **Plan** | 只读 + 规划契约（探索→建议→审批→执行） | 复杂任务规划 |
| **Interactive** | 默认模式，读自动放行，写/Shell 需审批 | 日常使用 |
| **Auto** | 完全信任模式，所有操作自动执行 | 可信自动化任务 |
| **Custom** | Interactive + 自动允许配置中的工具 | 高级用户 |

**模式转换流程：**

```
用户启动对话 → 默认为 Interactive 模式
    │
    ├─ 用户手动切换 → 立即生效
    │
    └─ 引擎请求升级 →
         Plan: AI 请求进入 Plan 模式
         Auto: AI 请求进入 Auto 模式
         (均需用户确认)
```

### 1.3 审批决策树

```
工具调用请求
    │
    ▼
┌──────────────────────┐
│  PermissionEngine     │
│  decision(outcome)    │
└──────────┬───────────┘
           │
    检查当前权限模式
    │
    ├─ Auto → 直接 ALLOW
    │
    ├─ Discuss → 拒绝写/SHELL/SEND 操作
    │
    ├─ Plan → 读 ALLOW，写拒绝（只能通过 propose_plan）
    │
    ├─ Interactive (默认)
    │    │
    │    ├─ RiskLevel.READ → ALLOW
    │    ├─ RiskLevel.WRITE → 检查 Override
    │    ├─ RiskLevel.SHELL → 检查 Override
    │    └─ RiskLevel.SEND → 检查 Override
    │
    └─ Custom → 按配置规则
           │
           ▼
    ┌──────────────────┐
    │  检查 Override    │
    │  (风险覆盖)        │
    └────────┬─────────┘
             │
    ├─ 配置了自动允许 → ALLOW
    ├─ 配置了拒绝     → DENY
    └─ 未配置         → NEEDS_USER (发出审批事件)
```

### 1.4 审批流程

当需要用户审批时，引擎：

1. **发出审批事件**：`approval.requested`，包含操作详情（工具名、参数、风险等级）
2. **等待响应**：`approver` 协程挂起当前 turn
3. **Surface 展示**：GUI/TUI 显示审批面板（操作内容、风险说明、建议）
4. **用户决策**：
   - **允许本次**：仅放行当前操作
   - **拒绝本次**：拒绝并通知引擎
   - **总是允许此类操作**：创建 Override 规则
5. **决策回传**：引擎恢复执行

```
引擎 → UI: approval.requested {
    tool: "file_write",
    params: {path: "/project/main.py"},
    risk: "write"
}
UI → 用户: 弹窗 "AI 想要写入文件 /project/main.py，允许吗？"
用户 → UI: 选择 "允许本次"
UI → 引擎: approval.resolved {decision: "allow"}
引擎 → 继续执行
```

---

## 2. 风险覆盖（Overrides）

**文件:** `coworker/overrides.py`

Overrides 是权限引擎的扩展点，允许用户对特定操作预设规则：

### 2.1 Override 类型

| 类型 | 说明 | 示例 |
|------|------|------|
| **工具级** | 对特定工具设置默认行为 | "自动允许 file_read" |
| **路径级** | 对特定文件路径设置 | "拒绝写入 /etc/passwd" |
| **模式级** | 按风险模式设置 | "允许所有 READ" |
| **时间级** | 按时间窗口设置 | "工作时间内允许 Shell" |

### 2.2 Override 优先级

```
1. 路径级 Deny (最高优先级)
2. 路径级 Allow
3. 工具级 Deny
4. 工具级 Allow
5. 模式级 Deny
6. 模式级 Allow
7. 权限模式默认行为 (最低优先级)
```

### 2.3 隐式 Override

用户每次的审批决策都会记录，当用户连续多次"允许本次"同类型操作后，系统会建议创建 Override 规则：

```
用户连续 3 次允许 file_read on /project/
→ 引擎建议: "检测到您多次允许此操作，是否设为自动允许？"
→ 用户确认 → 创建 Override {tool: "file_read", path: "/project/", action: "allow"}
```

---

## 3. 工作区信任

**文件:** `coworker/workspace_trust.py`

工作区信任机制控制 AI 对本地文件系统的访问范围：

| 信任级别 | 访问范围 | 场景 |
|---------|---------|------|
| **Trusted** | 完整读写权限 | 用户明确信任的目录 |
| **Constrained** | 仅限特定子目录 | 需要限制的项目 |
| **Untrusted** | 只读 + 无 Shell | 浏览性访问 |

工作区信任与权限模式正交：Trusted 工作区 + Interactive 模式 = 读写需审批；Trusted 工作区 + Auto 模式 = 完全自动。

---

## 4. 秘密管理

**文件:** `coworker/secrets.py`

连接器密钥和 API Token 存储在本地加密的文件系统中：

| 特性 | 说明 |
|------|------|
| **存储位置** | `state_dir/secrets/` |
| **加密方式** | 系统级密钥派生（macOS Keychain / 系统加密 API） |
| **作用域** | 全局（所有工作区共享） |
| **管理** | 通过 GUI 或 CLI 配置 |

**安全设计原则：**

1. 密钥在传输中加密（TLS），在存储中加密（本地加密）
2. 引擎读取密钥时注入到工具上下文，不写入对话历史
3. 连接器 Token 通过 OpenWorker Cloud 中继时端到端加密
4. 日志中自动脱敏密钥信息

---

## 5. 审计日志

**文件:** `coworker/audit.py`

所有 AI 操作记录审计日志：

```python
@dataclass
class AuditEntry:
    timestamp: datetime
    session_id: str
    agent_type: str
    tool_name: str
    tool_params: dict       # 脱敏后的参数
    risk_level: RiskLevel
    decision: str           # allow / deny / user_approval
    user_id: str | None
    result: str             # success / failed / denied
```

审计日志存储在 `state_dir/audit/`，支持导出和分析。

---

## 6. 无人值守与安全

**文件:** `coworker/unattended.py`, `coworker/selfwake.py`

### 6.1 无人值守模式

AI 可以在用户不在场时自主工作：

| 模式 | 行为 | 安全措施 |
|------|------|---------|
| **Scheduled** | cron 触发的定时任务 | 仅执行预配置的任务类型 |
| **Event-driven** | 消息/事件触发 | Approval 规则沿用最后设置的模式 |
| **Self-wake** | AI 主动发起（如监控到异常） | 默认 Plan 模式，不能写文件 |

### 6.2 自唤醒安全

**文件:** `coworker/selfwake.py`

AI 可以主动唤醒自己执行任务（如定时检查服务状态），但受限于：

- 只能使用 READ 等级的工具
- 不能执行 Shell 命令
- 不能发送外部消息（除非明确配置）
- 执行结果通过 Inbox 通知用户

---

## 7. 权限与连接器安全

### 7.1 连接器权限

每个连接器可以单独配置权限：

| 配置 | 说明 |
|------|------|
| `read` | 允许读取该服务的消息和数据 |
| `send` | 允许通过该服务发送消息 |
| `manage` | 允许修改该服务的配置 |

例如：允许 AI 读取 Slack 消息但禁止发送回复。

### 7.2 MCP 工具权限

MCP 工具的权限可以细粒度控制：

```toml
[mcp.servers.db-tool.permissions]
"query_database" = "auto"        # 自动允许
"write_database" = "approval"    # 需要审批
"drop_table" = "deny"            # 拒绝
```

---

## 8. Web 取址防 SSRF（URL 守卫）

**文件:** `coworker/web/guard.py`

`web_fetch` 和 `browser_read_url` 直接使用模型挑选的 URL，而模型的输入被视为**不可信**（它读网页、邮件和 Slack 消息，这些都是"数据而非指令"）。恶意页面可能诱骗 agent 去抓 `http://169.254.169.254/` 或 `http://127.0.0.1:11434/`，把只读调研工具变成对本机网络位置的探针——且 `web_fetch` 是 `requires_approval=False`，不会有审批弹窗出现。

**防护策略：**

| 机制 | 实现 | 位置 |
|------|------|------|
| **范围阻断** | 拒绝 loopback、RFC1918/私有空间、link-local（含云元数据端点 `169.254.169.254`）、保留/组播段，以及 **CGNAT**（`100.64.0.0/10`）——Python 的 `is_private` 漏掉它，而 Tailscale 会把内网主机分在这个段 | `guard.py:37` |
| **逐跳校验** | **每次重定向都检查地址**，`get_checked` 手动遍历最多 5 跳重定向——否则公开 URL 302 到 loopback 是绕过此过滤器的标准方式 | `guard.py:97` |
| **全 A 记录检查** | 多个地址中**任一**落进被阻范围即拒绝——防止"双 A 记录"名同时有公网和私网记录时被钻空子 | `guard.py:53` |
| **IPv4-mapped 还原** | `::ffff:127.0.0.1` 按其所承载的 v4 地址判属 | `guard.py:71` |

`browser_automation.py` 的 `open_url` 复用同一 `check_url`（`browser_automation.py:340`），针对模型提供 URL 做防御纵深（此路径有审批门控，故只是第二道防线，且不逐跳校验浏览器内部重定向）。

**未覆盖**：DNS rebinding——token 在守卫处与客户端连接时各解析一次，`~0` TTL 记录可能在两次之间变化；彻底封掉需连接级 IP 钉扎，逐跳校验是廉价且常用的 90% 方案。

---

## 9. 多会话隔离

SessionManager 确保安全边界：

| 隔离维度 | 说明 |
|---------|------|
| **会话隔离** | 每个 Session 有独立的 TurnEngine 实例 |
| **工作区隔离** | 不同工作区的文件/Shell 互不干扰 |
| **记忆隔离** | GLOBAL 记忆共享，WORKSPACE 记忆隔离 |
| **权限隔离** | 每个 Session 可有不同的权限模式 |

---

## 10. 代码索引

| 想了解什么 | 看哪个文件 |
|-----------|-----------|
| 权限引擎核心 | `coworker/permissions.py` |
| 风险等级定义 | `coworker/risk.py` |
| 风险覆盖规则 | `coworker/overrides.py` |
| 工作区信任 | `coworker/workspace_trust.py` |
| 密钥管理 | `coworker/secrets.py` |
| 审计日志 | `coworker/audit.py` |
| URL 守卫（防 SSRF） | `coworker/web/guard.py` |
| 浏览器自动化守卫 | `coworker/connectors/browser_automation.py` |
| 无人值守模式 | `coworker/unattended.py` |
| 自唤醒安全 | `coworker/selfwake.py` |
| 交互记录 | `coworker/interactions.py` |
| 未路由消息 | `coworker/unrouted.py` |
