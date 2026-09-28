# 变更记录

> 本解读对 DeepSeek Harness（`dsh`）的跨版本变更汇总；按版本倒序。项目无官方 changelog，故基于上游 git 代码变动（tag 间 diff / release commit / commit message 聚合）生成。各版本条目的"相比上一版"描述即对应某次增量解读落盘时更新的内容。

## [0.1.7-rc.2]（2026-09-24 落盘）

> 依据上游 `dsh-v0.1.5-rc.2 → dsh-v0.1.7-rc.2` git diff（3650 commits、8945 个文件、约 +1349K/−158K）比对生成；中间跨 `0.1.6-alpha.1`（09-15）、`0.1.6-alpha.2`（09-17）、`0.1.7-alpha.1`（09-22）、`0.1.7-alpha.2`（09-22）、`0.1.7-rc.1`（09-23）五版，另有 **0.1.5-rc.3（09-22）** 作为 0.1.5 线上的回移发布。
>
> **本轮范围**：第一遍完整重写 [desktop-app.md](desktop-app.md) 与 [deployment.md](deployment.md) 的桌面章节、新增 [desktop-zero-dependency.md](desktop-zero-dependency.md)；第二遍对全库做程序化重锚定（只改行号、不改论断，受跨度守卫与锚点唯一性守卫约束），**545 条引用被修正**，全库 `verify-references.mjs` 通过数由 811 升到 1356；第三遍把剩下的 340 条失配引用（182 MISMATCH / 158 UNVERIFIED）逐条读码改写，**全库 29 份子文档、1870 条引用全部通过**（`OK=1870 WEAK=0 MISMATCH=0 UNVERIFIED=0`）。详见 [README.md](README.md) 末节。

相比 **0.1.5-rc.2** 的主要变更（桌面端三条旧结论被反转 + 一轮外壳补全）：

**桌面传输面：无端口自研管道 → 回环 HTTP + 协议代理（反转）**
- 删除 fd3/fd4/fd5 分帧字节管道与 13 字节帧头（magic `0x44534833`），`desktop-host/config/desktop.cordis.patch.yml` 一并删除；Host 改为监听回环端口（默认 `19387`，与 Web 的 `3080` 独立）并回报带鉴权信息的 URL，壳侧用 `dsh-app://` 承担静态资产、把 API 流量代理过去。`DESKTOP_HOST_PROTOCOL_VERSION` 由 3 升到 4，控制面只留无 body 的 Node IPC 消息。决策笔记明言这笔交换 "gives up the portless guarantee"。see [desktop-app.md](desktop-app.md)、[deployment.md](deployment.md)。
- 代理侧新增三条显式规则：请求头重写、按 `WITHHELD_RESPONSE_HEADERS` 丢弃连接级响应头、插件包路径强制 `no-store`；`open-in-app` 与自动目录选择因此**回到桌面**（旧版因"桌面不提供 `webServer`"而禁用）。see [desktop-app.md](desktop-app.md)。

**桌面运行时：随包上游 Node → Electron 自己当 Node（反转）**
- `runtimeResources()` 返回值由 `{ node, pnpm, seed }` 变为 `{ node, nodeBin, pnpm, dsh }`，`node` 指向 Electron 可执行文件自身；启动 Host 依赖 `ELECTRON_RUN_AS_NODE=1` + `electronFuses: { runAsNode: true }`，并用 `--expose-internals` 补上 Electron 缺的内部 ESM loader 符号。see [desktop-zero-dependency.md](desktop-zero-dependency.md)。
- Node/pnpm/Python 版本改由 **primary-runtime 载荷**承担（Node 24.21.0 / Python 3.12.14，覆盖 `win-x64`/`mac-arm64`/`mac-x64`/`linux-x64`/`linux-arm64`），原生能力继续由预编译 Node-API addon 提供——用户机器仍不需要装 Node、pnpm 或 Python。

**桌面安装模型：seed 事务化安装 → 构建期物化 + 完整性清单（反转）**
- `apps/desktop/src/seed-store.ts`、`scripts/prepare-seed.ts`、`scripts/macos-seed-store.ts` 全部删除，"16 个未压缩分片 + 离线 pnpm store"这一层随之消失；改为构建期物化 `app.asar/dsh` 生产依赖树并写 `desktop-runtime.json` 逐文件清单，启动只做校验。
- profile staging/journal/rollback 激活事务下线（就地维护可写 profile），`profile-core-cleanup` 逻辑删除；`macOS 重签名 seed 内每个 Mach-O` 的重签名机制一并删除。

**桌面外壳补全（新增能力）**
- **托盘与退出语义**：`tray.ts` 常驻托盘、关窗默认转隐藏（Windows 首隐藏有一次持久化确认）、退出前向 Host 发 `quit-inspection`（2 秒上限，超时按"有活跃任务"保守处理）、`quit-confirmation.ts` 平台化按钮顺序。see [desktop-app.md](desktop-app.md)。
- **欢迎窗与账号**：`welcome-api.ts`/`welcome-window.ts` 承载"未登录且无 API key"的冷启动路径（只加载包内 HTML、禁导航与开窗）；Platform 文档用独立 WebContentsView，分区键按 `sha256(origin+userId)` 派生（未登录退化为内存分区）。
- **双更新通道**：普通更新（用户授权，下载单飞、必须等到 `update-downloaded`）+ **强制更新**（服务端策略，`DesktopMandatoryUpdatePolicy` 带 interval/backoff+jitter，取不到策略时保留上次拦截决策）；更新通道由 semver 推导改为固定 `nightly`，非预发布构建另传 `latest.yml` 别名。
- **Windows 目录式安装器**：`installer-directories.nsh` 的 `dshStageApplication`/`dshPromoteDirectories` 用「staging 目录 + 同卷两次 rename」替换逐文件覆盖（一次 staging 约 617 MB / 11735 个文件），失败回滚；`.mjs` 只做 `replaceOnce` 精确注入。
- **签名流水线工程化**：Windows 签名硬件调用从 379 次降到 17 次（产物签名缓存）、macOS 公证支持并发提交与 dotenv 凭据配置、`signIgnore` 跳过已预签名的 unpacked 子树避免二次签名。

**入口与打包路径迁移（破坏性调整）**
- electron-builder 配置由 `apps/desktop/electron-builder.config.mjs` 迁到 `apps/desktop/scripts/electron-builder-config.mjs`；`desktop-host` 包的 `files` 收窄为仅 `lib/index.js`（私有包，不再导出 overlay）。
- 会话格式、工具管道、沙箱、客户端等其他子系统的窗口内变更，已在本轮第三遍中逐条复核并更新（见下节），不再有「停留在旧基线」的子文档。
**全库增量复核：从「行号漂移」推进到「事实变更」（本轮新增）**

第二遍的程序化重锚定之后，剩下的 340 条失配引用（182 MISMATCH / 158 UNVERIFIED）全部逐条读码改写，29 份子文档归零：`verify-references.mjs` 现报 `OK=1870 WEAK=0 MISMATCH=0 UNVERIFIED=0 TRAVERSAL=0`。改写过程中确认了十处「机制级」变化，除上文桌面四条反转之外还有：

- **执行世界换血**：`packages/e2b/{e2b,fs-e2b,subprocess-e2b}` 退役，远端执行改由 `packages/ssh/{ssh,fs-ssh,subprocess-ssh,sandbox-ssh}` 承担；`ctx.sandbox.confine` 变为异步并接受可选 `signal`。see [subprocess.md](subprocess.md)、[sandbox-capability-seams.md](sandbox-capability-seams.md)。
（[.agents/notes/implemented/simplification/2026-09-11-remove-e2b-providers.md:17-17]()、[packages/sandbox/sandbox/src/index.ts:177-177]()）
<!-- anchor: The repository excludes the E2B sandbox owner, filesystem provider and subproces --> <!-- anchor: abstract confine(argv: readonly string[], policy: SandboxPolicy, signal?: AbortS -->
- **会话格式走到 v4（已定稿、未发布）**：writer 常量升到 4、catalog 组装改名 `sessionFormatCatalogOptions`；工具结果从 user 消息里解出来升成独立的 `tool` 角色消息，来源改受控 `kind`、未知标签命名空间化为 `plugin:<type>`；finalization 记录 4 而 release 记录仍是 3。see [session-format-migration.md](session-format-migration.md)、[session.md](session.md)。
（[packages/core/session/src/types.ts:89-89]()、[packages/session/session-format-catalog/src/generated.ts:16-30]()、[packages/session/session-format-v3-to-v4/src/tool-role.ts:36-65]()）
<!-- anchor: export const SESSION_FORMAT_VERSION = 4 --> <!-- anchor: export const sessionFormatCatalogOptions: SessionFormatCatalogOptions = { … sessionFormatV3ToV4, --> <!-- anchor: export function liftToolResult(event: SessionFormatEvent): SessionFormatEvent { … ...(isError === undefined ? {} : { isError }), -->
- **subprocess 多了一条 fd 7 控制通道**：新增 Definition 常量 `SUBPROCESS_CONTROL_FD = 7`，parent 侧由 `control-spawn.ts` 用一条 Socket 承载控制面；`subprocess-local` 同时拆出 `output.ts` 与 `shell-activity.ts`。see [subprocess.md](subprocess.md)。
（[packages/subprocess/subprocess/src/control.ts:7-7]()）
<!-- anchor: export const SUBPROCESS_CONTROL_FD = 7 -->
- **LLM 收敛到 Messages 协议**：`llm-deepseek` 变成纯协议库（凭据拆成两个插件），工具结果升为 `tool` 角色消息、新增 developer 角色与 tool-addition / tool-removal 块；`reasoning_content` 回放下线，改原生 thinking + signature。see [llm.md](llm.md)。
（[packages/llm/llm/src/message.ts:138-147]()）
<!-- anchor: /** Shared immutable fields of every conversation message. */ … readonly source: MessageSource -->
- **工作区文件**：`readAll` / `readRelated` 并入 `readBytes(path, { range?, baseFile? })`，二进制结果改走 Connection multipart（原生 `Uint8Array`）；`changes` 变成真 `fs.watch`（本地 Chokidar）并新增 `watch-unsupported`——「外部改文件不产生帧」的旧结论被反转。see [workspace-files.md](workspace-files.md)。
（[packages/api/workspace-files/README.md:50-50]()、[packages/api/workspace-files/README.md:58-58]()）
<!-- anchor: Pass the required options object as `{}` for a complete file under `maxFileBytes --> <!-- anchor: `changes` is a `stream` Remote accepting only a target path. The Host derives th -->
- **preset 与热更新**：`packages/preset/agent-presets` 拆成 `agent-preset` + `agent-preset-registry`，`agent.cordis.yml` 下线、改成 bundle patch 里插一条 `@deepseek-ai/dsh-agent-preset` 声明；`watchUserPatches` 与 `Profile.patchReload` 删除，热重放搬进 base 树的 `hmr` 行。see [architecture.md](architecture.md)。
（[packages/bundle/web-app/presets/minimal.patch.yml:1-8]()、[packages/boot/hmr/src/index.ts:229-235]()）
<!-- anchor: # Agent preset minimal: one `@deepseek-ai/dsh-agent-preset` declaration inserted … id: minimal --> <!-- anchor: const patches = readProfilePatches('dsh', profile) … for (const filename of patchFiles) await this.watchConfig(filename, () => refres -->
- **MCP 桥接面扩大**：resources 由兄弟包 `mcp-resources` 承担（prompt 模板与 resource subscriptions 仍不支持）；上一版自建的「重复游标即失败」判环守卫整体删除，改由 SDK 拥有分页。see [skill-mcp.md](skill-mcp.md)。
（[packages/mcp/mcp-client/README.md:209-209]()、[packages/mcp/mcp-client/README.md:80-80]()）
<!-- anchor: - **Resources are read on demand** — shipped profiles provide the [shared resour --> <!-- anchor: - The SDK owns discovery pagination and its page limit. A discovery failure pres -->
- **agent-loop / session 两处机制搬家**：inbox 投影注册从 `ReactLoopInbox` 构造函数上移到插件装配；「崩溃闭合」抽成通用的 `openTurnClosers(events, cause)` 并新增 `forked` 因由。see [agent-loop-runtime.md](agent-loop-runtime.md)、[session.md](session.md)。
（[packages/core/agent-loop/src/index.ts:362-365]()、[packages/core/session/src/repair.ts:63-162]()）
<!-- anchor: // Register only after every config validation above has passed, so a … ctx.sessionProjections.register(inboxProjectionDefinition) --> <!-- anchor: export function openTurnClosers(events: readonly SessionEvent[], cause: OpenTurn … closers.push({ type: 'turn/end', seq: SessionSeq(seq++), time, data: { turn: ope -->
- **分发形态与家族地图**：`PUBLIC_EXPERIMENTAL_PACKAGE_DIRECTORIES` 白名单删除，改为「默认公开 + 一份当前为空的私有例外名单」，agent-team 的生成式 Remote API 下线；`packages/` 现 54 家族 / 312 包，`packages/code-runtime` → `packages/ptc-runtime`，`packages/fs/tool-present` → `packages/deliverables/tool-present`。see [agent-teams.md](agent-teams.md)、[architecture.md](architecture.md)。
（[scripts/experimental-package-policy.ts:10-15]()、[packages/ptc-runtime/README.md:6-6]()、[packages/deliverables/tool-present/src/index.ts:40-42]()）
<!-- anchor: export function isPublicExperimentalPackageDirectory( … && !privateDirectories.includes(directory) --> <!-- anchor: # ptc-runtime/ — PTC execution capability family --> <!-- anchor: description: 'Declare existing files as final deliverables for the user. ' … + 'prefer your final response when that suffices. The user opens the current fil -->
- **已知盲区**：校验程序只认 `[path:start-end]()` 形式，KB 里另有 87 处「反引号短引用」（`agent-loop-runtime.md` 38 处、`tools.md` 27 处）尚未纳入程序校验，见 [README.md](README.md) 末节。


## [0.1.5-rc.2]（2026-09-11 落盘）

> 依据上游 `dsh-v0.1.5-rc.1 → dsh-v0.1.5-rc.2` git diff（4 commits、324 个文件，绝大多数是快照/黄金文件复算）比对生成。

相比 **0.1.5-rc.1** 的主要变更（功能面只有一条）：

**Web 前端 UX**
- 把已合入 master 的**反馈对话框与文件交付卡片打磨**回移到 0.1.5 发布线（`feat(web): backport feedback and file refinements to 0.1.5`）：统一 markdown 预览组件与注释、图表预览按图片内容定尺寸、交付卡片按 workspace 相对路径显示。无接口或格式变更。see [web-client.md](web-client.md)。

## [0.1.5-rc.1]（2026-09-11 落盘）

> 依据上游 `dsh-v0.1.5-alpha.2 → dsh-v0.1.5-rc.1` git diff（17 commits、349 个文件、约 +1.1K/−0.7K）比对生成。

相比 **0.1.5-alpha.2** 的主要变更（收窄为模型目录 + 侧栏观感）：

**LLM 模型目录**
- **Chat Completions 默认路由切到 DeepSeek V41 Flash**（`feat(llm): default Chat Completions to DeepSeek V41 Flash`），同时**保留 V4 系列并存**（`retain V4 models alongside V41 Flash`）、**恢复 `deepseek-v4-flash-vision-exp` 视觉条目**（`restore V4 Flash Vision Exp catalog entry`）。see [llm.md](llm.md)。
- webhook 冒烟与 Cordis 夹具的模型路由随之钉死（`test: keep webhook smoke on the shipped model route`）。

**Web 前端 UX**
- 侧栏 guide 起始页与统计 pill 去噪、文档预览滚动区跟随当前预览（`fix(client): keep preview scrollports current`）。see [web-client.md](web-client.md)。

## [0.1.5-alpha.2]（2026-09-11 落盘）

> 依据上游 `dsh-v0.1.5-alpha.1 → dsh-v0.1.5-alpha.2` git diff（262 commits、1151 个文件、约 +27K/−10K）比对生成。

相比 **0.1.5-alpha.1** 的主要变更（主题是**客户端外壳与文件呈现的成品化**）：

**Web 客户端 / 侧栏（本次主体）**
- **侧栏 tab 体系补齐**：last-tab 关闭规则与折叠展开按钮（`feat(sidebar): last-tab close rules and collapsed expand button`）、**live tab 标题**（文件类型图标 + 路径去歧义）、**图片预览**、**全局侧栏面板**（`feat(client): add global sidebar panels`）。see [web-client.md](web-client.md)。
- **可扩展文档预览**：`feat(client): add extensible document previews and sidebar defaults`，PDF 连续滚动、分页文件 tab 与保留阅读状态（`feat(textpreview): add paged file tabs and retained reader state`）。
- **Markdown 栅栏预览扩展（已回滚，未进入本版）**：Mermaid（`feat(ui): preview Mermaid diagrams in Chat code blocks`）→ Graphviz / SVG / HTML（`feat(client): preview Graphviz, SVG, and HTML Markdown fences`），并适配应用主题；随后被 PR #3901 `Revert "feat(web): preview Mermaid, Graphviz, SVG, and HTML code fences"` **完整回滚**（净删约 2346 行，含两个 Agent Note、e2e 用例、第三方许可清单与依赖）。HEAD 上代码栅栏只做语法高亮，**没有**图预览。see [web-client.md](web-client.md)。
- **共享文件类型图标 + 统一链接语言**：`feat(primitives): add file-type icon set and shared link icon`、`feat(client): add shared file type icons`。
- **交付呈现**：不可变交付物的**下载卡片**（`feat(web): present immutable file deliveries with download cards`）、artifact card 的**原生文件操作**（`feat(web): add native file actions to artifact cards`）、按 workspace 相对路径显示（`fix(web): show artifact paths relative to the workspace`）。see [workspace-files.md](workspace-files.md)。
- **反馈入口收敛**：`/feedback` 与 Dislike 合并为**同一个带分类的反馈对话框**（`feat(feedback): open one feedback dialog from /feedback and Dislike, with categories and a toast`）。
- **dockkit 打磨**：strip/chip 呈现、关闭策略与 room reading（`feat(dockkit): refine strip and chip presentation, close policy, and room reading`）。

**工具集**
- **`str_replace_editor` 从 minimal profiles 移除**（`feat(minimal): remove str_replace_editor from minimal profiles`），至此它已从 base / sdk / minimal 三个默认组合全部退出、改为**显式 opt-in**；同版本内的 `refactor: remove str_replace_editor`（彻底删除代码）被 `Revert "refactor: remove str_replace_editor"` 回滚，**包仍在 HEAD**。see [tools.md](tools.md)。

**子代理**
- **父方拥有的 child catalog 记录与观察**（`feat(subagent): record and observe parent-owned child catalogs`），并抽出**可持久化 chunked-list 工具**（`refactor(subagent): extract persistent chunked list utility`）。see [subagent.md](subagent.md)、[data-model.md](data-model.md)。

**分发与发行**
- **实验性 Agent Teams 包正式随发行版发布**（`release: publish experimental Agent Teams packages`，此前为 private）。see [agent-teams.md](agent-teams.md)。
- minimal 形态的持久 bash 输出对齐一次性 shell 契约（`feat(minimal): align persistent bash output with the one-shot shell contract`）。

**接口 / 行为**
- **投影视图收敛为 all-or-none**：同一批提交先把 `projectionStates` / `projectionStateKeys` 这类"逐视图挑选"的 API 加出来（`refactor(session-query): return all projection states by default`），随后又由同批的 `refactor(subagent): expose catalog through existing projection views` 收回，**HEAD 上只剩 `projectionMode: 'all' | 'none'` 两档**（默认 `'all'`，位于 session-query 的 observation 层）；llm-pi-ai 启动恢复复用 settings 校验。

## [0.1.5-alpha.1]（2026-09-11 落盘）

> 依据上游 `dsh-v0.1.3-alpha.2 → dsh-v0.1.5-alpha.1` git diff（563 commits、2042 个文件、约 +68K/−11K，为本次增量中最大的单版）比对生成。

相比 **0.1.3-alpha.2** 的主要变更：

**Session 格式 V3（破坏性，本次核心）**
- **system prompt 提升为 surface node zero**：`refactor(session): represent the system prompt as surface node zero` + `feat(session-format): migrate V2 prompts into V3 system surfaces` + `feat(session): add identity V2-to-V3 migration and writer skeleton`。V3 成为 0.1.5-alpha.1 发布的第 3 代格式，V2 的 prompt 记录被迁移进 V3 的 system surface。see [session-format-migration.md](session-format-migration.md)、[system-prompt.md](system-prompt.md)。
- 迁移正确性加固：`fix(session): audit every historical content carrier before V3 migration`、`fix(session): retain unknown required events for vocabulary validation`、`fix(session): canonicalize V3 envelopes through adjacent migration`。see [session-format-migration.md](session-format-migration.md)。

**桌面形态（新增）**
- **Electron 桌面端落地**：`feat: electron 打包`、`feat: mac code sign & notarize`、`feat: windows build & sign`、`feat: optimize ipc perf`；新增 `apps/desktop` 与 `apps/desktop-host` 两个包（Host 从 CLI 包中移出），target-specific 打包与自动更新路径（`feat(desktop): update packaging and auto-update paths for target-specific builds`、`enhance auto-update configuration and add upload scripts`）。see [desktop-app.md](desktop-app.md)。

**原生 addon 家族**
- **`native/landlock-run` → `native/system`（node-addon-system）**：`refactor(native): rename package family to node-addon-system`、`move Landlock workspace to native/system`、`expose Landlock through its capability subpath`，并新增**预编译 Node-API flock**（`feat(native): add prebuilt Node-API flock support`）。see [desktop-app.md](desktop-app.md)、[sandbox-capability-seams.md](sandbox-capability-seams.md)。

**工作区文件能力面（新增）**
- **resource 寻址 + 双面文件 API**：`feat(workspace-path): define Session and absolute file resource addresses`、`feat(workspace-files): add dual-face file API and Host-resolved resources`、`feat(remotes): expose workspace file operations to the Client`。
- **有界字节区间读取下沉到 fs seam**：`feat(fs): define bounded byte-range reads` + `feat(fs-local): implement bounded local byte windows` + `feat(fs-e2b): implement cancellable sandbox byte windows`，`/api/file` 随之只认 `bytes` range、按扩展名 allowlist 提供。see [workspace-files.md](workspace-files.md)。

**Web 客户端**
- **Client resource registry 与保留订阅**（`feat(resources): add Client resource registry and retained subscriptions`）；**侧栏 tab 导航 / 注入信息 / fullscreen 外壳**（`feat(sidebar): add tab navigation, injected information, and fullscreen shell`）；**可逆 dock 引擎**（`feat(dockkit): add reversible docking engine and pointer interactions`）；**惰性 workspace 文件树 tab**（`feat(sidebar-files): add lazy workspace file tree tabs`）；**交付物经 Sidebar resources 打开**（`feat(deliverables): open produced files through Sidebar resources`）；响应式右列（`feat(layout): add responsive right column and width concessions`）。see [web-client.md](web-client.md)。

**Agent loop / Inbox**
- **Inbox 由服务改为 durable projection**：`refactor(agent): remove inbox service` + `refactor(agent): back Inbox with a durable projection` + `refactor(agent-loop): own inbox projection in loop`；**system prompt 变更在 capable route 上追加为消息**（`feat(agent-loop): append system prompt changes on capable routes`）。see [agent-loop-runtime.md](agent-loop-runtime.md)、[session.md](session.md)。

**网络代理**
- **所有出站请求走配置的代理**（`feat(net): route every outbound request through the configured proxy`），http-proxy 收敛为 util 库。see [networking-proxy.md](networking-proxy.md)。

**其他**
- `feat(visualizer): add dormant Host capability`（休眠状态的可视化 Host 能力面）——随后被 PR #3778 `Revert "feat(visualizer): add dormant Host capability"` 回滚，**未进入本版**，HEAD 无 `packages/visualizer`；`feat(cli): create profiles from shipped templates`（`--from-default-profile`）；compaction 把**派生 system head 作为消息重放**（`refactor(compaction): replay the derived system head as a message`）。see [memory-compaction.md](memory-compaction.md)、[deployment.md](deployment.md)。

## [0.1.3-alpha.2]（2026-09-11 落盘）

> 依据上游 `dsh-v0.1.3-alpha.1 → dsh-v0.1.3-alpha.2` git diff（316 commits、1339 个文件、约 +35K/−9K）比对生成。

相比 **0.1.3-alpha.1** 的主要变更：

**新增能力 / 行为**
- **模型切换可观察**：`feat(agent): announce model switches`——切换模型在会话流里留下可回放的通知帧。see [agent-loop-runtime.md](agent-loop-runtime.md)。
- **人机 inbox 控制对齐**（`feat(subagent): align human inbox controls`）：面向人类操作者的 inbox 交互与状态收敛。see [subagent.md](subagent.md)。
- **Web UI 打开本地应用**：`feat(workspace): open the workspace in local apps from the web UI (#3409)`。see [workspace-files.md](workspace-files.md)、[desktop-app.md](desktop-app.md)。
- **`str_replace_editor` 退出默认工具集（0.1.3-alpha.2）**：`feat(base): remove str_replace_editor from default tools` + `feat(sdk): disable str_replace_editor by default`；0.1.5-alpha.2 再从 minimal profiles 移除（`feat(minimal): remove str_replace_editor from minimal profiles`）。**注意代码面并未删除**——同一版本内的 `refactor: remove str_replace_editor` 随即被 `Revert "refactor: remove str_replace_editor"` 回滚，`packages/fs/tool-str-replace-editor` 在 HEAD 仍存在，只是从各默认组合退出、改为**显式 opt-in**（base README 明说 "remains available as an explicit opt-in"）。see [tools.md](tools.md)。

**子进程子系统（本版大量收敛）**
- native containment 基线重建（`refactor(subprocess): restore native containment baseline`），runner 执行面/pid liveness/Windows settlement 逐项收敛（`derive runner execution plane` / `pass runner pid to liveness probe` / `simplify Windows runner settlement`）。see [subprocess.md](subprocess.md)。

**Session format / 上下文**
- 引入**流式迁移 stage**（`refactor(session-format): introduce streaming migration stages`，0.1.5-alpha.1 再接 `feat(session-format): migrate V2 prompts into V3 system surfaces`）。see [session-format-migration.md](session-format-migration.md)。
- session-reference 默认预算按所选模型定尺寸、截断捕获转 spill 以便检索（`fix(session-reference): size default budget from selected model`、`fix: spill truncated session-reference captures for retrieval`）。see [session.md](session.md)。
- 持久化读取：历史读取先准备再发布、冻结读取直接移交进 restore（`perf(session-persistence): prepare historical reads before publication`、`perf(session): transfer frozen persistence reads into restore`）。see [data-model.md](data-model.md)。

**系统提示词**
- **persona 前缀保留在系统提示首、cwd 移到后缀**（`fix(system-prompt): keep model persona prefix and place cwd in suffix`）；**环境事实排到可复用指令之后**（`fix(system-prompt): place environment facts after reusable instructions`）——为提示缓存命中率服务。see [system-prompt.md](system-prompt.md)。

## [0.1.3-alpha.1]（2026-09-11 落盘）

> 依据上游 `dsh-v0.1.2-rc.1 → dsh-v0.1.3-alpha.1` git diff（328 commits、1442 个文件、约 +56K/−23K）比对生成。

相比 **0.1.2-rc.1** 的主要变更：

**破坏性变更**
- **会话格式进入"已发布世代 + 相邻迁移链"**：`feat(session)!: add released format migration` + `feat(session)!: embed assistant streams in format v2`——格式版本从"只写当前版、旧版拒绝"演进为**不可变世代 + N→N+1 转换链**，并把 assistant 流内嵌进 v2。see [session-format-migration.md](session-format-migration.md)。
- **持久化层改为 handle 化 seam + 生命周期拥有的写路径**：`refactor(session-persistence)!: handle-based seam with a lifecycle-owned write path`，JSONL 后端新增**跨进程写所有权租约**（`feat(session-persistence-jsonl): cross-process write-ownership lease`）。see [data-model.md](data-model.md)、[session.md](session.md)。

**新增能力 / 行为**
- **同会话消息编辑（已回滚，未进入本版）**：`feat(session, agent, web): support same-session message editing` 曾在开发窗口内合入（PR #3452），随即被 `Revert "feat(session, agent, web): support same-session message editing"` 完整回滚（PR #3459），且回滚先于本版打标签——两个提交都在 `dsh-v0.1.2-rc.1..HEAD` 区间内但净效果为零，HEAD 不存在编辑已落库消息的 API。仍然存活的只是 **pending 队列项编辑**（`updateQueue` → `agent.inbox.replace/remove`）。see [session.md](session.md)、[agent-loop-runtime.md](agent-loop-runtime.md)。
- **附件从"图片专用"泛化为通用文件**：`feat(attachment): store and project generic files` + `refactor(attachment): own prompt admission on service` / `isolate file upload service`。see [attachments.md](attachments.md)。
- **实时 assistant 流帧**：`feat(agent): emit live assistant stream frames`。see [agent-loop-runtime.md](agent-loop-runtime.md)。
- **出站代理**：`feat(net): route every outbound request through the configured proxy`（http-proxy 收敛为 util 库，最终 API 为**四函数**：`installProxyFromEnvironment` / `proxyRouteFor` / `proxyEnvironmentForChild` / `clearedProxyEnv`）；usage telemetry 的 OTLP 通道被明确排除在外（`revert(session-telemetry-otel): leave telemetry on its own transport`——telemetry 是唯一"丢了不损失任何功能"的出站通道）。see [networking-proxy.md](networking-proxy.md)。
- **Python SDK 支持 macOS x64 runtime wheels**（`feat(python): support macOS x64 runtime wheels`）。see [deployment.md](deployment.md)。
- **会话控制器承接文件**：`feat(session-controller): admit files across command flows`。see [remote-controllers.md](remote-controllers.md)。
- **Agent Teams 统一 steer 消息**（`feat(agent-team): unify messages on steer`）。see [agent-teams.md](agent-teams.md)。

**Web 前端 UX**
- 附件混合呈现统一（`feat(ui-attachment): unify mixed attachment presentation`）、`read_image` 结果直接渲染成图片卡片（含嵌套调用）、文件随提交生命周期流式上传（`feat(conversation): stream files through submission lifecycle`）。
- 可点击链接语言统一（`feat(client): unify clickable-link language with link alias and category glyphs`）；技能候选用共享模糊名排序（`feat(web): rank skill candidates with the shared fuzzy name ranker`）。see [web-client.md](web-client.md)、[skill-mcp.md](skill-mcp.md)。

**注**：**0.1.4 版本号被整段跳过**——发布序列为 `0.1.3-alpha.2` → `0.1.5-alpha.1`，无 0.1.4 tag 或 release commit。

## [0.1.2-rc.1]（2026-09-03 落盘）

> 依据上游 `dsh-v0.1.2-alpha.4 → dsh-v0.1.2-rc.1` git diff（跨 alpha.5、rc.1 两个 release commit，代码面约 1300 行含测试夹具、全落在 storage-domain / storage-json / session-projection-cache）比对生成。

相比 **0.1.2-alpha.4** 的主要变更（代码面高度聚焦，主题单一：**storage 层跨版本读兼容 + 失效派生数据抢救**）：

**Schema / 数据演进（本次唯一主体）**
- **`DomainSpec` / `KvUnitDescriptor` 新增 `compatibleVersions` 跨版本读兼容白名单**：per-record 布局读侧从「只认当前版本戳」放宽为接受 `[current, ...compatibleVersions]`（`parseRecord` 签名从单版本改版本集、`acceptedStamps` 汇总）；owner 声明"旧版记录我的当前 schema 也认"，靠把旧版缺的字段声明为 optional 背书；`single` 布局仍 exact-version 拒绝。写永远盖当前版本，兼容只发生在读侧、绝不自动迁移。see [data-model.md](data-model.md)。
- **legacy 整文件 bootstrap 增加版本门槛**：bootstrap 前校验 legacy 整文件 stored unit version 在接受集内，未背书的旧版本遗留文件原样保留、整 unit 读作空——避免"迁移 owner 从未担保的 stale 缓存 → 盖章当前版本 → 域层 schema 失败"。
- **`DomainSpec.invalidRecords: 'backup-and-skip'` + `KvUnit.backupRecord` 失效记录抢救**：open 校验某条记录 zod schema 失败时，声明该策略且后端能移文档（per-record）即把 `<key>.json` rename 成 `<key>.json.bak.<YYYYMMDDHHmm>`、记日志、按缺席续开；权威数据与无 per-record 文档的后端（single/行存）保持整体拒绝。语义分界 = 权威数据拒绝、可丢弃派生数据跳过。
- **projcache 域 spec 演进出第一个消费方**：`session_projcache` 域声明 `version:5 + compatibleVersions:[3,4] + invalidRecords:'backup-and-skip'`——升级后的 home 不必等每个 session 重开就继续服务缓存列表投影；`checkpointIdentity` 谱系位 `isSeeded`/`inheritedEventCount` 变可选，缺席经 `compatibleVersions` 放行的 v3/v4 记录读作 unseeded，`identityMatches` 对缺席 `?? false`/`?? 0`。
- `tool-cordis` API catalog 同步刷新（`DomainSpec`/`KvUnit`/`KvUnitDescriptor` 声明文本随接口演进）。

**注**：rc.1 与 alpha.5 之间无独立 feature 级移除。rc.1 之后的 master 曾短暂合入又回滚 `feat(session-message-edit)`，不属于本版本代码面。

## [0.1.2-alpha.4]（2026-09-01 落盘）

> 依据上游 `dsh-v0.1.1-rc.2 → dsh-v0.1.2-alpha.4` git diff（跨 4 个 alpha 版本、含 4 个 release commit）比对生成。

相比 **0.1.1-rc.2** 的主要变更：

**破坏性变更**
- **SQLite 会话持久化后端整体移除**（`refactor(session)!: remove SQLite persistence backend`）：`packages/session/session-persistence-sqlite` 目录删除，事件日志收敛为 JSONL 单一 shipped 后端；Schema 17 打包物理行、`SCHEMA_VERSION` pragma 拒绝随之失效。KV 层 `storage-sqlite` 保留。see [data-model.md](data-model.md)。
- **session 事件 seq 与 log offset 品牌分离**（`refactor(session)!`）：`SessionSeq`（事件身份）/ `SessionLogOffset`（间隙/前缀/读偏移）两个 branded number；`SessionHeader.seedLength` 存储坐标改 `isSeeded: boolean` + body 侧 `inheritedEventCount`；v0 JSONL header 字节兼容（缺席/零/非零 seedLength 解码规则保持）。see [session.md](session.md)。
- **apiproxy RPC 移除**（`refactor(apiproxy)!` ×2）：settings/credentials RPC 与 directory-picker RPC 整体删除（约 −839 行），由 Typert Remote 控制器取代。see [remote-controllers.md](remote-controllers.md)。

**新增能力 / 行为**
- **Typert Remote 控制器体系**：`@Remote`/`@RemoteScope` 声明式暴露 + 构建期 `typert-generator` 生成 Host/Client 双契约；`api-remotes` 挂载、`api-gateway` 经 Connection `/api` 路由分发；`session`/`workspace`/`settings`/`credentials` 四控制器落地；事件经 `API_REMOTE_FORWARDED_EVENTS` 白名单转发。see [remote-controllers.md](remote-controllers.md)。
- **Webhook 子系统**：`ctx.webhookRuntime` 无状态 fire-and-forget 规则运行时 + `webhook-github` 签名 HTTP 适配器；外部投递经规则变成普通 root Session（`source.kind: 'webhook'` 溯源）。see [webhook.md](webhook.md)。
- **Inspector（experimental，private）**：CDP 跨 Host/浏览器双域检视，CDP 状态隔离在 Worker + 收集器快照投影。see [inspector.md](inspector.md)。
- **WebWorker（experimental，private）**：浏览器 Worker 跑整棵插件树（packer 打 VFS 镜像、运行时零编译器、嵌套 Worker 子进程）。see [webworker.md](webworker.md)。
- **storage-json per-record 布局**：`single`（整文件）与 `per-record`（`<root>/<unit>/<table>/<key>.json` + global.json）双布局，legacy 整文件首开自动 bootstrap；投影缓存域随之每会话一文档。
- **projection-cache 冷读播种**：`coldSnapshot` 从身份校验的缓存行 seed fold、尾重放补足、写回刷新（fail-soft）；`checkpointIdentity` 加入 `isSeeded`/`inheritedEventCount` 谱系位，`row.seq` 用 `SessionSeqCursor`。
- **sdk-minimal profile 与 SDK/ACP application bundle**：独立双工具 SDK profile（显式排除 base/Web/settings/凭据/telemetry 等）+ Python/TS SDK 与 ACP 的预置组合分发。
- **web fetch 默认暴露**：base/headless/sdk 三面默认暴露 web fetch，web 形态要求一次性 fetch 审批（one-shot approval）。
- **subagent 模型路由**：`tool-subagent` 新增 model-selection 状态与设置面，可为 Codex/Claude Code provider 配置模型，模型路由可经 DSH SDK 透传。
- **session 持久化模型选择**：`session` 持久化并共享模型选择目录；session-query 新增 shared projected observations。
- **DeepSeek 适配层**：增量 session 日志上传、插件包元数据上传、会话上传评审反馈（session-log-deepseek / deepseek-llm-api-extensions）。

**Web 前端 UX**
- **全会话 turn rail**：session-turn-outline 整日志 turn 大纲投影 + ui-chat 侧栏 load-and-jump 深度分页 + settled-response 预算预览。
- **lexical composer**：ui-conversation 用词法编辑器取代 textarea 栈；连接恢复指示器；usage/time 双 pill；大滚动性能优化（流式几何采样节流、帧节流发布等）。

**接口 / API 变更**
- `api-gateway`（旧 apiproxy）的 `ctx.remote.<namespace>` 调用面取代手工 RPC；`ctx.webhookRuntime` 为新增 capability；`ctx.inspector` 为 experimental 注册面。
- session 构造器对 seed 显式要求（含空 seed + cut 0），`ownEvents()`/`isOwnSeq()` 隐藏继承 cut 比较。

**移除 / 回滚**
- **format migration decoder pipeline 引入后回滚**：`feat(session): add format migration decoder pipeline`（cc9ab200c7）在 alpha 周期内被 `Revert`（211e6939e3），不进入 alpha.4。
- `session-persistence-sqlite`、`tool-subagent-report`、旧 apiproxy 的 credentials/settings/rpc 实现被移除。

## [0.1.1-rc.2]（2026-08-23 落盘）

> 依据上游 `dsh-v0.1.1-rc.1 → HEAD(0.1.1-rc.2)` git diff（含 release commit `aa6c361a97`）比对生成。

相比 **0.1.1-rc.1** 的主要变更：

**新增能力 / 行为**
- **图片/附件管线统一为"确定性归一化 + 对象投影"**：上传图片落库前被归一化为固定 8-bit sRGB、去元数据与动画（仅"干净的源"字节级透传，其余重编码 + 降采样到 `DEFAULT_NORMALIZED_IMAGE_MAX_DIMENSION=2048` 长边）；`AttachmentStore` 新增抽象 `readImageRequest` 投影能力，按模型路由的像素/字节预算派生确定性请求图（`RequestImageAttachment`，本地实现带磁盘 cache 与 variant 身份）。see [attachments.md](attachments.md)。
- **master 与 Files API 请求管线合并（file-id 首选、base64 兜底）**：DeepSeek 适配器请求图片默认走 `file`（Files API 上传复用），超限量化移除；仅当 Files 解析失败或预算超限才回退内联 base64，并对 provider 拒绝/过期 file-id 做失效重试。
- **新增 DeepSeek Files API transport**：完整 `/files` 端点客户端（upload/list/retrieve/delete）、带过期时间与并发去重的上传索引、对 `dsh-` 前缀自有文件按 `reclaimOldestOwned` 回收以应对配额。
- **新增 `prepareCall` 两阶段调用 API**：adapter 从单一 `stream` 演进为 `prepareCall(provider, model)` 产出"冻结的模型信息 + stream 函数"，使同一模型/连接快照绑定、可独立重放；跨 adapter 一致（llm-deepseek / llm-pi-ai 均导出）。see [llm.md](llm.md)。
- **Files 与 stream 超时解耦**：新增独立 `filesApiTimeoutMs`（默认 60s）配置，与 stream idle 超时（300s）分离；Files 解析失败自动回退 base64。
- **`read_image` 工具上报降采样维度与坐标换算提示**：读图结果携带 `originalDimensions`，model-facing envelope 给"multiply coordinates by …"。

**Schema / 数据**
- `SCHEMA_VERSION` **未变化**（仍为 17），session 持久化格式保持稳定。

**接口 / API 变更**
- **权限默认继承回滚**：整体回滚此前合入的 `reuseWorkspaceBlank`（空白会话复用继承默认权限），`session.create` 移除 `reuseWorkspaceBlank`+`sessionId` 组合参数，sessions-port 接口与 UI 权限展示同步收敛。

**移除 / 修正**
- `packages/image` 代码收敛进 `packages/attachment`，attachment-local 新增 `normalization.ts` / `request-image.ts` / `compression-limiter.ts` / `encoding.ts`。
- WebP alpha 兼容修正：sharp 从全不透明 WebP 省略 alpha 平面时不再误判失败。

## [0.1.1-rc.1]（2026-08-23 落盘）

> 依据上游 `dsh-v0.1.0-rc.8 → dsh-v0.1.1-rc.1` git diff（含 release commit `3ec5e8f8c4`）比对生成。

相比 **0.1.0-rc.8** 的主要变更：

**新增能力 / 行为**
- **凭据系统从"环境变量引用"升级为"可交互授权 + 持久化记录"**：新增 `packages/credentials/authorization` 子系统（`AuthorizationPrompt` 的 text/secret/select 三态人机提问、`AuthorizationMethod` 描述 OAuth/API-Key），provider 可注册授权 flow；新增持久化 `CredentialRecord`（`ApiKeyRecord | GrantRecord`，grant payload 对 harness 不透明）、两段式 `CredentialKey = <scope>/<id>` 键空间。see [credentials.md](credentials.md)。
- **凭据文档升级为版本化布局**：`$DSH_HOME/.credentials.yaml` 引入 `DOCUMENT_VERSION=1` 结构化布局（`{version, refs, records}`），boot 时 `migrateFlatDocument()` 一次性迁移旧平面前置布局，写回持有跨进程 writer lock、外部编辑热发布。pi-ai 提供商现在能真正"登录"（OAuth/device-code 桥接进通用授权平面）。
- **Session 投影重写为"状态 + 客户端视图"两段式**：`ProjectionDefinition` 改为声明式纯 fold（`stateSchema` 校验持久化状态、可选 `wire` 提供客户端视图）；新增 host-only 单元（状态不进客户端快照仍照常 checkpoint）；新增统一 `checkpoint()` / `restoreFloor()`，跨 boot 稳定快照。see [session.md](session.md)。
- **DeepSeek 视觉模型进目录**：新增模型 `deepseek-v4-flash-vision-exp`（`inputModalities: ['text','image']`），模型目录新增 `inputModalities` 校验，vision 多模态落地。see [llm.md](llm.md)。

**接口 / API 变更**
- `apiproxy` 的 `session.create` 支持 `reuseWorkspaceBlank` 复用空白会话（配套权限预设方向 `origin` 追踪：`default | selection | inferred`）——**注：该能力在下一个版本 rc.2 被回滚**。
- `web server` 新增结构化 `webserver/index-inject` 事件与 `collectIndexInjections()`，取代裸字符串 `tapIndex` 注入；前端缺失路径返 404、非 GET/HEAD 返 405。

**安全 / 沙箱**
- bwrap profile 新增 `--unshare-pid`，把沙箱进程放进独立 PID 命名空间，关闭 process-root 逃逸。see [sandbox-capability-seams.md](sandbox-capability-seams.md)。

**Web 前端 UX**
- 子代理会话头部升级为"血缘切换器"（`SubagentHeaderLineage`，openChild/refresh/catalog）；同回合重试耗尽渲染"终态 turn 错误"行；宽表按列数自适应 + hover 滚动条；`ask_user_question` 支持多行回答（上限六行）。
