# 变更记录

> 本解读对 Tutti 的跨版本变更汇总；按版本倒序。描述清楚相关变动即可：若上游有官方 changelog，参照它并结合代码变动生成；没有则基于代码变动（git diff / release notes / 代码快照比对）生成。

## [v0.2.30]

更新日期：2026-08-28。依据上游 git log / release notes（v0.2.11 → v0.2.30 区间，约 731 个 commit）。

### 新增能力

- **Tutti Agent**：内置免费 agent，Early Access 期间免费（README FAQ），并配套 `docs/architecture/tutti-agent-readiness-bootstrap.md` 就绪引导文档。
- **远程 Connector MCP**：新增 `docs/architecture/connector-remote-mcp.md`，远程 connector 经 `tsh-server` Connector MCP Gateway 执行，Tutti 侧用 MCP `2026-07-28` 无状态 HTTP Client（`RemoteMCPClientFactory`），本地聚合 MCP Server `connector` 对 Agent 依旧只暴露本地视角。
- **presence / room 多用户实时**：新增 room-scoped realtime client（`feat(presence)`），为 Tutti · VM 多用户协作铺路。
- **全局 activity client**：新增 global activity client（`feat(agent-activity)`）。
- **Windows 桌面支持落地**：README 提供 Windows x64 桌面下载；release 侧新增 Certum Windows 签名自动化（`feat(release): automate Certum Windows signing`），并新增 `docs/architecture/desktop-windows.md`。

### 架构与重构

- **Connector 模块大重构**（8/19，`refactor(connector): converge daemon and renderer ownership`）：
  - `services/tuttid/service/connectormarket/` 迁移为 `services/tuttid/service/connector/{market,mcp}`，daemon 与 renderer 所有权收敛。
  - 原 `broker.go` 的 4 个命令（`connector.available`/`connector.skills`/`connector.skill.read`/`connector.invoke`）收敛为 `command_provider.go` 的 `connector.available` 单命令；连接器执行转由原生 MCP / CLI 面拥有。
  - `docs/architecture/connector-market.md` 扩展重构为 `connector.md`（820 行），并新增 `connector-remote-mcp.md`。
  - 实现宿主相关逻辑下沉到 `packages/connector/runtime/implementationhost/`（route projection / mcp registry / credential authorization 等）。
- 仓库改名：`github.com/anomalyco/tutti` → `github.com/tutti-os/tutti`（原地址已 404）。

### 交互与接入变化

- Connector 能力面变化：`connector.skills` / `connector.skill.read` / `connector.invoke` 不再作为 daemon CLI 命令暴露（`connector.invoke` 现返回 `ErrCommandNotFound`）。
- Connector 安装/授权体验迭代：授权对话框、catalog 分类（动态 market categories）、verified remote archives 安装、`published HTTPS icons` 硬性要求（`feat(connector)!: require published HTTPS icons`）。

### Schema 与数据演进

- `service/eventstream/catalog.go` 主题目录新增 `account.userpresence.updated`、`agent.side.updated` 等 topic。

### 解读文档

- 新增 `acp-protocol.md`：ACP 协议深入关注面（双向 JSON-RPC 契约、标准 ACP 适配器、会话/turn 生命周期、权限交互、setup 探测与安全边界）。
- 重写 `connector-market.md`：按 v0.2.30 connector 重构后架构更新（共享 `packages/connector` + 宿主注入、命令面收敛、远程 MCP）。
- README / app-ecosystem / agent-extensions 行号引用同步至当前 HEAD；仓库地址修正为 `tutti-os/tutti`。

### 移除与废弃

- 删除 `docs/architecture/connector-market.md`（重构为 `connector.md` + `connector-remote-mcp.md`）。
