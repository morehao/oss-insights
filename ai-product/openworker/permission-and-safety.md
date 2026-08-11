# OpenWorker 权限与安全模型深度分析

## 1. 权限引擎（PermissionEngine）

**文件:** `coworker/permissions.py`

PermissionEngine 是 OpenWorker 安全体系的核心——它不是简单的"允许/拒绝"门控，而是一个**基于风险等级的决策引擎**。

### 1.1 风险分类

**文件:** `coworker/risk.py`

每个工具调用在执行前都被分配到一个**风险类别（RiskClass）**，它是"内在副作用类别"，驱动权限门控（以及后续无人值守的 Inbox 路由）：

| 类别 | 枚举值 | 含义 | 判定方式 |
|------|--------|------|---------|
| **读** | `RiskClass.READ` | 无副作用，永远放行 | 纯读工具 |
| **写本地** | `RiskClass.WRITE_LOCAL` | 改动工作区——路径作用域 + 模式门控 | 名单 `WRITE_TOOLS` |
| **执行** | `RiskClass.EXEC` | 运行命令——模式门控 | 名单 `SHELL_TOOL="run_shell"` |
| **外部** | `RiskClass.EXTERNAL` | 机器外的副作用——无人值守 Inbox 钩子 | aisuite metadata `requires_approval`→外部 |

```python
class RiskClass(str, Enum):
    READ = "read"  # no side effects — always allowed
    WRITE_LOCAL = "write_local"  # mutates the workspace — path-scoped + mode-gated
    EXEC = "exec"  # runs commands — mode-gated
    EXTERNAL = "external"  # side effects off the machine — the unattended Inbox hook

WRITE_TOOLS = {"write_file", "replace_in_file", "apply_patch", "apply_unified_diff"}
SHELL_TOOL = "run_shell"
```

`classify()`（`risk.py:39-53`）是**单一判定入口**：优先级为 `用户 override → 名单基础表（_BASE）→ aisuite requires_approval→EXTERNAL → 兜底 READ`。`is_consequential()`（`risk.py:56-58`）定义"非纯读即需要权限引擎注意"。**判据不是文档旧版的 SEND 等级**——连接器和 MCP 工具主要通过 EXTERNAL（外部副作用）被归类，而不是单独的"SEND"等级。【`coworker/risk.py:18-58`】

> **演进说明(源码 HEAD 01b6f83)**：早期文档用 `RiskLevel(READ/WRITE/SHELL/SEND)` 四级枚举，与真实源码不符——真实是 `RiskClass(READ/WRITE_LOCAL/EXEC/EXTERNAL)` + 工具名单，风险从"硬编码名集合"改为"工具声明的属性，`classify` 统一读取"。发送类操作归入 EXTERNAL 语义。

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

真实决策全在 `PermissionEngine.evaluate()`（`permissions.py:120-178`），按固定顺序短路。注意名字是 `evaluate` 而非旧文档的 `decision`，模式枚举是 `Mode.DISCUSS/PLAN/INTERACTIVE/AUTO/CUSTOM`：

```
工具调用请求
    │
    ▼
PermissionEngine.evaluate(tool_name, arguments, metadata)
    │  classify() → risk (用户override > 名单 > requires_approval > READ)
    │
    ├─ 模式为 read-only (DISCUSS/PLAN) 且 consequential → [拒] "<mode> is read-only"
    │
    ├─ 写工具且 path 不在可写 root（_under_writable_root）→ [拒] "path is not writable"
    │
    ├─ 非 consequential（纯 READ）→ [放行] "low risk"
    │
    ├─ Auto 模式 → [放行] "full access"
    │
    ├─ Interactive/Custom
    │    ├─ shell: 命令匹配 _command_allowed / session_allow_commands → [放行]
    │    ├─ 工具在 session_allow_tools（非 connector）→ [放行] "tool allowed for session"
    │    └─ task standing rule 精确 target → [放行] "allowed by standing rule: tool→target"
    │
    ├─ Custom 且工具在 auto_allow_tools → [放行] "auto-allowed by config"
    │
    └─ 其他 → [需用户] Decision(needs_user=True)
```

**路径作用域**：写工具的 `path` 参数会经 `_under_writable_root` 校验（`permissions.py:204-214`），必须落在声明为可写的 root 内——即使是 Auto 模式之前也先做此检查。

**shell 白名单防注入**：`_command_allowed`（`permissions.py:216-238`）先拒绝含 shell 操作符（`; & | > < \` $( (`，见 `_has_shell_operators`）的命令，再用 `shlex.split` 做 **argv 前缀精确匹配**（`git status` 匹配 `git status -s`，但绝不匹配 `git statusfoo` 或 `git status && rm -rf ~`）。

### 1.4 审批流程

当 `evaluate` 返回 `needs_user=True` 时，引擎走审批流。审批结果类型是 `ApprovalOutcome`（`engine.py:30-34`），不是简单的 allow/deny：

```python
class ApprovalOutcome(str, Enum):
    ONCE = "once"             # 仅放行本次
    ALWAYS_TOOL = "always_tool"      # 会话内放行此工具
    ALWAYS_COMMAND = "always_command"  # 会话内放行此命令（仅 shell）
    DENY = "deny"             # 拒绝
```

流程（`_authorize`，`engine.py:673-780`）：

1. `permissions.evaluate()` 判定；需要用户时**先发出 `PERMISSION_REQUIRED` 事件**（含工具名、参数、reason、category、standing_target），再审计记录（`engine.py:697-716`）
2. `await self._interruptible(self.approver(PermissionRequest(...)))` 挂起当前 turn 等用户（`engine.py:717-728`）
3. 用户决策：
   - **ONCE**：仅放行当前调用，不记状态
   - **ALWAYS_TOOL**：调 `permissions.allow_tool_for_session(tool_name)`，写入会话 allowlist，本会话内后续该工具直接放行（`engine.py:742-743`）
   - **ALWAYS_COMMAND**：调 `allow_command_for_session(command)`，仅对 shell 有效，本会话内该命令放行（`engine.py:744-747`）
   - **DENY**：拒绝，发 `TOOL_FINISHED: denied`（`engine.py:729-740`）
4. 决策回传，引擎恢复执行；未知工具在 `_authorize` 末尾被拒绝并追加 tool-error 消息（`engine.py:757-767`）

```
引擎 → 表面: PERMISSION_REQUIRED {
    name: "run_shell", arguments: {command: "git status --porcelain"},
    reason: "requires approval", category: "", standing_target: null }
用户: 选择处理（本次 / 此工具总允许 / 此命令总允许 / 拒绝）
用户 → 引擎: ApprovalOutcome.ALWAYS_COMMAND
引擎: allow_command_for_session("git status --porcelain")  # 本会话白名单
引擎 → 继续执行
```

> **审批结果持久化**：`ALWAYS_TOOL`/`ALWAYS_COMMAND` 产生的会话 allowlist 会被 `manager.py` 的 `_grants_of` 序列化保存（会话重建后恢复）；非 `ONCE` 的结果都落审计（`approval` 字段记录 outcome 值）。

---

## 2. 风险覆盖（Overrides）

**文件:** `coworker/overrides.py`

`RiskOverrideStore` 是用户的本地风险覆盖库——核心用途是**放宽 MCP 的保守默认**：每个 MCP 工具默认 `external`，信任某 server 的用户可把其只读工具标成 `read` 以停止门控。规则按工具名 glob 匹配，最具体者胜（`overrides.py:77-85`）。

### 2.1 覆盖机制

规则是一个 `pattern` + 一个 `RiskClass` 的映射（`overrides.py:23-27`）。`resolve` 遍历规则，用 `fnmatchcase` 匹配工具名，按 `_specificity`（字面非通配字符数，精确匹配 +1000，`overrides.py:29-33`）取最具体者：

| 概念 | 说明 | 示例 |
|------|------|------|
| **pattern** | 工具名的 glob（可用 `* ? []` 通配） | `"mcp__notion__*"` |
| **risk** | 覆盖后的 RiskClass | `read` / `external` / `exec` |
| **最具体胜出** | literal 字符越多越具体；精确匹配压过一切 glob | `mcp__notion__create_page` 精确规则 > `mcp__notion__*` |

```python
def resolve(self, tool_name: str) -> Optional[RiskClass]:
    best: Optional[RiskClass] = None
    best_score = -1
    for r in self._rules:
        if fnmatchcase(tool_name, r.pattern):
            score = _specificity(r.pattern)
            if score > best_score:
                best, best_score = r.risk, score
    return best
```

> **Evolution 说明(源码 HEAD 01b6f83)**：旧文档的"工具级/路径级/模式级/时间级"四类 Override 与真实实现不符——真实 `overrides.py` 只做**按工具名 glob 覆盖风险类别**，且**铁律是用户本地专属、persona/package 永不写它**（`overrides.py:7-9`）。路径级写文件防护由 `evaluate` 内的 `_under_writable_root` 承担（见 1.3），命令级白名单由 `_command_allowed`/会话 allowlist 承担，都不在这层。

### 2.3 隐式授权：会话 allowlist 与 standing rule

"记住并放行"这类隐式授权在真实源码里有**两种**机制，都不走 `overrides.py`：

- **会话 allowlist（审批"总是允许"）**：用户审批时选 `ALWAYS_TOOL` / `ALWAYS_COMMAND`，调用 `allow_tool_for_session` / `allow_command_for_session`（`permissions.py:181-186`）写入本会话的 `session_allow_tools` / `session_allow_commands`，供后续 `_command_allowed` / 工具 allowlist 检索（见 1.4）。生命周期到会话结束。
- **任务级 standing rule**：由自动化任务持有，规则绑定"工具 + 精确 target"（`standing_rule_candidate`，`permissions.py:62-80`），`evaluate` 里命中即自动放行并审计具体规则（`permissions.py:165-171`）——精确 target 绑定使自动化放行连接器工具也安全。**绝不用于 exec 风险**，且只从 external 风险工具提取候选。

> **演进说明(源码 HEAD 01b6f83)**：旧文档"连续多次允许后系统建议建 Override"的描述与实际不符——真实是审批 UI 直接以 `ApprovalOutcome` 三态（本次/此工具/此命令）反馈，无"自动检测连续放行并提示"的逻辑。

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

所有 AI 操作写入本地 SQLite `AuditStore`，真实 schema 是 `audit_events` 表（`audit.py:33-48`）：

| 字段 | 说明 |
|------|------|
| `timestamp` | 默认 `CURRENT_TIMESTAMP` |
| `session_id` / `agent` / `workspace` | 归属信息 |
| `connector` / `tool` | 操作主体（connector 由 `connector_for_tool` 反查） |
| `stage` | proposed / started / approval_requested / approval_resolved / finished / auto_allowed / interrupted |
| `status` / `approval` | 结果状态 + `ApprovalOutcome` 值 |
| `args` | **脱敏后的参数**（JSON，见下） |
| `result_preview` / `reason` / `resource` | 结果摘要 / 原因 / 定位资源（url/repo/issue_key 等） |

**脱敏（`_sanitize_args`，`audit.py:123-137`）**：含 `token/secret/password/api_key/access_token/bot_token/app_token/raw` 等键名 → `[redacted]`；`body/content/html` → `[redacted body]`；其余递归截断（列表取前 10、dict 取前 20 键、文本 500 字符）。`resource` 从 url/owner/repo/issue_key/page_id 等参数提取（`audit.py:152-169`）。`list()` 支持按 `session_id/connector/tool` 过滤、`id DESC` 排序。

> **演进说明(源码 HEAD 01b6f83)**：旧文档的 `AuditEntry` dataclass 字段（`risk_level`/`decision`/`user_id`）与真实实现不符——真实是 `audit_events` SQLite 表，列为上表，无 `risk_level`，改用 `stage`/`status`/`approval` 描述决策过程，脱敏规则也更强（键名与 body 双重防御）。审计与记忆、会话、自动化共同构成 OpenWorker 的持久化层，见 [`data-model.md`](./data-model.md)。

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
