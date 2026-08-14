# MCP 统一管理（Model Context Protocol）

> 仓库根：`/Users/morehao/Documents/study/ai/agent-product/AionUi`；引用为仓库相对路径。真正的"按 agent 能力注入/同步 transport"核心算法在 AionCore（Rust 后端），本稿只覆盖前端壳。凡断言跨到后端标注【待主 agent 验证】。

## 一句话定位

AionUi 把「每个 agent 各管各的 MCP 配置」收拢成一个统一清单：前端只负责**配置模型表达、HTTP 透传、连接测试/状态展示、一键导入**，配置存储、按会话/agent 注入 transport 的判定与结果回填全走 `/api/mcp/*` REST 接口由后端统一拥有——前端是"壳 + 门面"，后端是"大脑"。

## MCP server 配置模型

全部类型定义收敛在 `common/config/storage.ts`（前后端共享的单一事实来源）。四种 transport + 一个 server 结构：

```
[packages/desktop/src/common/config/storage.ts:609-622]() IMcpServer：enabled + transport + original_json + builtin
<!-- snippet: export interface IMcpServer {
  id: string;
  name: string;
  description?: string;
  enabled: boolean; // 是否默认启用（新会话默认勾选）
  transport: IMcpServerTransport;
  tools?: IMcpTool[];
  last_test_status?: 'connected' | 'disconnected' | 'error' | 'testing';
  last_connected?: number;
  created_at: number;
  updated_at: number;
  original_json: string; // 存储原始JSON配置，用于编辑时的准确显示
  builtin?: boolean;
} -->
```

关键字段设计（Why）：
- **`original_json`**：保留用户原始 JSON 配置供编辑"准确显示"——导入时不做破坏性规范化，避免回显失真。
- **`enabled`**：语义是"启用的 server 作为新会话默认装载"，而非"当前会话是否在用"。
- **`builtin`**：内置 server 前端据此隐藏编辑/删除入口。

transports：`stdio` = `command`/`args`/`env`（本地子进程）；`sse`/`http`/`streamable_http` = `url`/`headers`（远端）。配置键在 `storage.ts:101` 的 `mcp.config`，但注释明确"当前所有写入应走 aionCore-owned storage"（`storage.ts:90-91`）——前端这份只是遗留参照备份，真实存储由后端接管。

## 统一管理：前端全走后端 REST

前端唯一的"CRUD + 测试 + 认证"通道是 `mcpService`（`common/adapter/ipcBridge.ts:1049-1113`），全部 HTTP：`/api/mcp/servers`、`/api/mcp/agent-configs`、`/api/mcp/test-connection`、`/api/mcp/servers/*/toggle`、`/api/mcp/oauth/*`。三个关键信号：

1. **`/api/mcp/agent-configs`** — 前端拉取"各已装 CLI agent 自己现有的 MCP 配置"，是"同步各 agent 自带 MCP"的入口。
2. **`/api/mcp/test-connection`** — 连接测试。
3. **`/api/mcp/servers/*/toggle`** — 启用开关持久化在后端。

【待主 agent 验证】真正的"把 MCP server 注入具体 agent 会话、按 `mcpCapabilities` 判定 stdio/http/sse 哪种 transport 兼容"的算法在 AionCore Rust 侧。

### 会话级装载结果的旁证

前端从会话对象读取装载结果 `mcp_servers` / `mcp_statuses`，状态机 `'loaded' | 'failed' | 'unsupported'` 正是"sync compatible transports 后"的逐 server 结果：

```
[packages/desktop/src/common/config/storage.ts:627-634]() 会话级 MCP 装载状态：loaded/failed/unsupported
<!-- snippet: export type IConversationMcpStatusKind = 'loaded' | 'failed' | 'unsupported';

export interface IConversationMcpStatus {
  id: string;
  name: string;
  status: IConversationMcpStatusKind;
  reason?: string;
} -->
```

前端在会话渲染处透传这份后端快照（`ChatConversation.tsx:245-247`）。`unsupported` 即"该 server transport 不被当前 agent 支持"，是 README "syncs compatible transports" 的前端可见落点，但判定在后端。

ACP 侧 agent 能力声明定义在 `acpTypes.ts:39-43`（`AcpMcpCapabilities`），前端还提供从握手快照提取支持 transport 列表的 `getSupportedMcpTransports`（`agentTypes.ts:247-257`）——不过该函数目前无 renderer 调用点（消费逻辑主要在后端）。

### transport 宽进窄出：`streamable_http` 降级为 `http`

前端模型支持 4 种 transport（含 streamable_http），但向后端提交前归一为 `http`：

```
[packages/desktop/src/renderer/hooks/mcp/catalog.ts:43-52]() 前端把 streamable_http 归一为 http 提交给后端
<!-- snippet: const normalizeTransportForBackend = (transport: IMcpServerTransport): BackendMcpTransport => {
  if (transport.type === 'streamable_http') {
    return {
      type: 'http',
      url: transport.url,
      headers: transport.headers,
    };
  }
  return transport;
}; -->
```

`BackendMcpTransport = Exclude<IMcpServerTransport, { type: 'streamable_http' }>`（`catalog.ts:5`）——前端传输更宽、后端更窄，兼容性在边界收敛。`catalog.ts` 还合并内置 + 用户两来源并去重（`builtin:`/`user:` 前缀）。

### 一键导入：把各 CLI agent 自带 MCP 同步进统一清单

`OneClickImportModal.tsx` 是"统一管理"最直接入口——选择 Claude/Codex，拉取其 CLI 现有 MCP 配置，过滤不可导入项后批量并入：

```
[packages/desktop/src/renderer/pages/settings/components/OneClickImportModal.tsx:203-209]() 一键导入：拉取 CLI 现有 MCP 配置
<!-- snippet:   const handleImportFromCLI = async () => {
    setLoadingImport(true);
    try {
      const mcpConfigs = await mcpService.getAgentMcpConfigs.invoke();
      const selectedConfig = mcpConfigs.find((agentConfig) => agentConfig.source === selectedAgent);
      const allServers = (selectedConfig?.servers ?? []) as DetectedMcpServer[];
      setFetchedServers(allServers); -->
```

可导入性由后端返回的 `importable` + `import_skip_reason` 决定，skip 原因被本地规范化为受支持文案（Plugin-managed / Disabled / Needs authentication / Disconnected 等，`OneClickImportModal.tsx:27-47`）——这就是后端判定"该 transport 对当前 agent 不兼容/不可用"的暴露面。

## 调试工具：`debug:mcp` 悬空引用 + 真正的连接测试

根 `package.json:59-61` 声明了 `debug:mcp` / `debug:mcp:list` / `debug:mcp:validate` 三个命令，指向 `scripts/debug-mcp.ts`——但**全仓找不到该文件**，`docs/` 下也没有 MCP 调试文档。判定：这三个命令是**悬空引用**，属于 README/package.json 承诺与源码不符处。

前端实际的连接调试是 `testMcpConnection`（`ipcBridge.ts:1089-1109`），后端负责实际建立 stdio/SSE/HTTP 连接、枚举 tools、返回分级错误码。错误码在 `useMcpConnection.ts:43-112` 被翻译为可读信息，涵盖 `MCP_COMMAND_NOT_FOUND`（细分 node/bun/uv/python/deno 运行时）、`MCP_COMMAND_PERMISSION_DENIED`、`MCP_TIMEOUT`、`MCP_CONNECTION_FAILED`、`MCP_HTTP_ERROR`、`MCP_RPC_ERROR`、`MCP_PROTOCOL_ERROR`——结构化连接诊断体系：

```
[packages/desktop/src/renderer/hooks/mcp/useMcpConnection.ts:47-59]() MCP 失败码 → 用户可读文案（分运行时）
<!-- snippet:   switch (payload.code) {
    case 'MCP_COMMAND_NOT_FOUND':
      switch (details.runtime) {
        case 'node':
          return t('settings.mcpErrorNodeCommandNotFound', {
            command: details.command || 'npx',
            defaultValue: fallback,
          });
        case 'bun':
          return t('settings.mcpErrorBunCommandNotFound', {
            command: details.command || 'bunx',
            defaultValue: fallback,
          }); -->
```

测试结果的 tools 回填 server 的 `tools` 字段在 UI 展示（`useMcpConnection.ts:190-200`）。OAuth 型远端在测试前先做认证状态检查（`useMcpOAuth.ts:20-29` 只对 URL transport 取 url，stdio 跳过）。

## 内置 MCP 资源：如何打包提供

资源目录 `packages/desktop/src/process/resources/builtinMcp/`，含图片生成 MCP + 应用内浏览器 MCP 启动器 + CDP 桥。内置 server 的**注册**在进程启动迁移阶段注入后端（`runBackendMigrations.ts`）。

### 图片生成内置 server

`imageGenServer.ts` 是一个独立 stdio 进程，用官方 `@modelcontextprotocol/sdk` 起 server，向 agent 暴露 `aionui_image_generation` 工具：

```
[process/.../builtinMcp/imageGenServer.ts:13-14,41-45,129-130]() 真 stdio server，暴露 aionui_image_generation 工具
<!-- snippet: import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { z } from 'zod';

// Read provider config from environment variables
function getProviderFromEnv(): TProviderWithModel | null {
  const platform = process.env.AIONUI_IMG_PLATFORM;
  const base_url = process.env.AIONUI_IMG_BASE_URL;
  const api_key = process.env.AIONUI_IMG_API_KEY;
  const model = process.env.AIONUI_IMG_MODEL; -->
```

生成时直接从环境变量读 provider 配置（`AIONUI_IMG_PLATFORM`/`BASE_URL`/`API_KEY`/`MODEL`）——即 transport 的 `env` 是配置载体，不写死在 command 参数里。

### 内置 server 注册清单与默认启用策略

`runBackendMigrations.ts` 构建三个内置 server 并批量导入后端，默认启用策略（Why）：

- **chrome-devtools**：默认**禁用**（`enabled:false`，需本机装 npx 包）；
- **内置浏览器**（`aionui-browser`）：默认**启用**（零配置开箱即用）；
- **图片生成**：按用户是否在 Settings 选好图生模型决定（`enabled: config?.switch === true && resolution.ok`）。

统一原则：**越不依赖用户额外配置、越安全，越默认开**。内置 server 全部 `builtin: true`，前端据此隐藏编辑/删除。

### 应用内浏览器内置 server：动态端口与静态注册的解耦

`browserServer.ts` 是 transport 兼容的典型。chrome-devtools-mcp 只接受 `--browser-url` CLI 参数、不认环境变量，而 AionUi 的 CDP 端口是**动态**的。于是注册记录**永远指向中转脚本**，端口经环境变量传入，脚本运行时翻译成 `--browser-url` 再拉起真正的 chrome-devtools-mcp：

```
[packages/desktop/src/process/resources/builtinMcp/browserServer.ts:150-162]() 中转脚本：env 口 → --browser-url，stdio inherit 直通协议流
<!-- snippet: const spawnPlan = buildMcpSpawnCommand({
  platform: process.platform,
  version: CHROME_DEVTOOLS_MCP_VERSION,
  browserUrl,
});

const child = spawn(spawnPlan.command, spawnPlan.args, {
  // stdio 直通：这个进程只是转发者，MCP 协议流不能被中间层缓冲或改写
  // Pass stdio straight through: this process is a forwarder, and the MCP
  // protocol stream must not be buffered or rewritten in between.
  stdio: 'inherit',
  env: process.env,
}); -->
```

配套权衡（文件注释浓墨重彩）：固定版本 `0.16.0` 而非 `@latest`（首启要联网解析 + 安全/确定性优先）；无端口/口令就退出而非回退开独立 Chrome（避免 agent 操作用户看不见的浏览器）；Windows `.cmd` spawn 用 `cmd.exe /c` 而非 `shell:true`（规避 CVE-2024-27980 EINVAL + Node24 DEP0190 警告与 cmd 元字符注入）。

## 关键权衡（Why）

| 权衡 | 取舍 |
|------|------|
| 配置"集中在后端、前端只做门面" | 所有 agent 共享一份 server 清单（一处配置处处生效）；代价是单点依赖后端，离线/无后端时前端 MCP 管理不可用 |
| transport 宽进窄出 | 前端宽（4 种）、后端窄（归一 http），兼容性在边界收敛 |
| 内置 server 默认启用按"零配置可用性 + 安全"折中 | 越不用配置越安全越默认开 |
| 动态发现与静态注册解耦（browserServer 中转层） | 避免"改端口就改数据库 + 多实例写串"竞态，环境变量是配置载体 |
| 导入保留"可导入性分层" | 统一管理≠强制统一；按性能/认证/状态过滤，避免破坏性虹吸 |

## 亮点与问题

- **亮点**：`original_json` 保留字面输入、`builtin` 隐藏受管项、`last_test_status` 区分"最近一次检测 vs 会话内实时状态"都体现产品细粒度；`browserServer` 中转层是"动态配置 + 静态注册"解耦的漂亮案例；结构化 MCP 错误码体系覆盖到运行时细分。
- **问题**：`debug:mcp` 三个命令是悬空引用（承诺与源码不符）；`getSupportedMcpTransports` 定义但无调用点，部分能力待接线或依赖后端返回。

## 与其他模块的协同

- ACP agent 能力声明（[`multi-agent-acp.md`](multi-agent-acp.md)）定义 `stdio/http/sse`，是"按能力注入 transport"的判定契约。
- 会话级 `mcp_statuses` 存于 `conversations.extra`（[`data-model.md`](data-model.md)）。
- 内置 MCP 注册发生在 `runBackendMigrations.ts`（架构的迁移阶段）（[`architecture.md`](architecture.md)）。

> 【待主 agent 验证】Rust 后端 `/api/mcp/*` 与"按 mcpCapabilities 注入/同步 transport + `unsupported` 判定"的核心实现；`scripts/debug-mcp.ts` 是否在他仓存在。

## 主要源码文件

- `common/config/storage.ts` — 配置模型单一事实来源
- `common/adapter/ipcBridge.ts` — mcpService（/api/mcp/* 透传）
- `renderer/hooks/mcp/catalog.ts` — transport 归一 + 多源合并去重
- `renderer/hooks/mcp/useMcpConnection.ts` — 连接测试 + 错误码翻译
- `packages/desktop/src/renderer/pages/settings/components/OneClickImportModal.tsx` — 一键导入
- `process/resources/builtinMcp/{imageGenServer,browserServer,cdpBridge}.ts` — 内置 MCP
- `process/utils/runBackendMigrations.ts` — 内置 server 注册
- `common/types/platform/acpTypes.ts` — agent 能力声明