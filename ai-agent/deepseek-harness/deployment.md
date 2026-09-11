---
type: 项目解读
title: "部署形态：Profile/Bundle/Patch 分层与 web/headless/desktop 三形态"
description: "DeepSeek Harness（dsh）部署形态解读——Profile/Bundle/Patch 三层启动组合、web 与 headless 与 desktop 三形态分发矩阵、内置 bundle 与随附 profile 模板（--from-default-profile）、electron-builder 打包与签名公证、Python SDK 单文件运行时与平台 wheel 矩阵、以及代理与环境事实的启动期注入。"
tags: [deepseek-harness, dsh, deployment, profile, bundle, patch, headless, desktop, electron, python-sdk, packaging]
---

# 部署形态：Profile/Bundle/Patch 分层与 web/headless/desktop 三形态

> 子文档之一，聚焦用户如何安装、运行、扩展 dsh。覆盖 `apps/cli`（`dsh` bin）、`apps/desktop` + `apps/desktop-host`（Electron 壳与私有 Host 进程）、`packages/boot/app-boot`、`packages/bundle/{base,web-app,headless,sdk-app,sdk-minimal,acp-app}`、`python/`（SDK 单文件运行时）与 `native/landlock-run`。整体哲学见 [README](README.md)；本文上承 [data-model.md](data-model.md)（数据落地）、回扣 [architecture.md](architecture.md)（部署即组成插件树）；桌面的打包/签名/自动更新细节下放 [desktop-app.md](desktop-app.md)，工具 roster 的裁剪见 [tools.md](tools.md)。

> **本版增量更新（0.1.2-rc.1 → 0.1.5-rc.2）**：相比上一版，部署面新增三条线索。**(1) 第三种发行形态**：Electron 桌面端并入分发矩阵，但它**不新造 bundle**——桌面端把 `web` 那一套 `base + web-app` 组合原样装进独占 profile，再用一层私有 patch 把传输面从 HTTP 换成进程内管道（细节见 [desktop-app.md](desktop-app.md)）。**(2) profile 的创建入口显式化**：新增 `--from-default-profile`，把"模板 profile 首次自动初始化"这一隐式行为升级为可指名创建、独占占位的显式动作。**(3) 分发单元继续下沉**：Python SDK 的单文件运行时补上 macOS x64 目标，并把 SEA 入口 dispatcher 收回到 `python/sdk-runtime/` 自有打包面；`sdk-minimal` 组合与 `minimal` preset 同步移除 `str_replace_editor`、把持久 shell 的输出契约对齐一次性 shell。

部署层是对外唯一安装/运行入口：profile（命名组合）、bundle（分发单元）、patch（行为切片）把「一切皆插件」落地为可一键跑起、又可零侵入定制。

## 项目中的角色：去掉它则用户无法启动、也几乎无法扩展

`dsh` 命令行产品对外唯一的安装与运行入口就是一个部署层。没有任何其他面向用户的启动面：库包（core/session/agent 等）只是可组合部件，只有 profile 启动把它们的 Cordis 配置行列成一颗真实可跑的树；`dsh plugin` 是"安装插件"这一动作的实现者；bundle patch 揭示了"插件如何带着自己的配置行一起分发"。

启动入口分派到三个模式：`profile`（跑指定 profile）、`plugin`（管理插件）、`dump-config`（诊断组合树，不启动），三者都落在 `dsh` 这个 bin 下（[apps/cli/src/bin.ts:31-60]()
<!-- anchor: switch (invocation.mode) { … throw new Error(`dsh: unhandled invocation mode ${JSON.stringify(invocation)}`) -->）。`web` 是 `--profile web` 的硬编码别名，所以"启动 Web 形态"本身就是部署层的一个子命令（[apps/cli/src/args.ts:175-187]()
<!-- anchor: const web = program.command('web').description('boot the web profile (alias of - … resolved = resolveBoot(web, 'web', options, args) -->）；0.1.5 新增的 `--from-default-profile` 复用同一套解析（[apps/cli/src/args.ts:146-146]()
<!-- anchor-base64:Lm9wdGlvbignLS1mcm9tLWRlZmF1bHQtcHJvZmlsZSA8bmFtZT4nLCAnaW5pdGlhbGl6ZSBhIG5ldyBjdXN0b20gcHJvZmlsZSBmcm9tIGE= -->）。同版本还加了一条**反向门禁**：CLI 显式拒绝 `--profile desktop`，把那个 profile 判定为 Electron 壳独占（[apps/cli/src/args.ts:68-70]()
<!-- anchor: function rejectElectronProfile(program: Command, profile: string): void { … program.error('error: profile "desktop" is managed exclusively by the Electron a -->）——部署层在这里第一次承认"存在一个自己不该碰的 profile"。

任何一个普通开发者安装一个第三方 bundle 后想真正跑起来，路径上必经两件事：`reconcilePlugins` 把安装的依赖按 `dsh.bundle` 声明写回 `dsh.profile.bundles`（[apps/cli/src/plugin.ts:59-90]()
<!-- anchor: function reconcilePlugins(before: ProfileManifest, profileDir: string): void { … writeProfileManifest(profileDir, after) -->），以及 `composeProfile` 按层组合 patch（[apps/cli/src/profile-boot.ts:226-243]()
<!-- anchor: async function composeProfile( … return { profile, bundlePatches, homePatches, overlays: composedOverlays } -->）。没有这个部署层，一切皆插件只是一个不可运行的类型系统。

```mermaid
flowchart TD
    U["用户/CI"] -->|npx @deepseek-ai/dsh web| CLI["bin: lib/bin.js"]
    U -->|pnpm dsh web| CLI
    U -->|Electron 壳拉起私有 Host| DTOP["apps/desktop-host"]
    CLI -->|parseDshArgs| ARGS["args.ts<br>web别名 / --profile<br>/ --patch / 模板创建"]
    ARGS -->|profile 模式| RUN["profile-boot.ts<br>runProfile"]
    ARGS -->|plugin 模式| PLG["plugin.ts<br>pnpm转发 + reconcile"]
    ARGS -->|dump-config 模式| DMP["dump-config.ts<br>组合树打印"]
    RUN --> PREP["prepareProfile loadProfile"]
    DTOP --> PREP2["loadProfileDirectory<br>profile 目录由壳独占"]
    PREP --> MAN["profile package.json<br>dsh.profile.bundles"]
    PREP2 --> MAN
    MAN --> BUND["bundle package.json<br>dsh.bundle.patch"]
    BUND --> BUNDP["该 bundle 的<br>cordis.patch.yml"]
    RUN --> COMP["composeProfile<br>各层合成"]
    DTOP --> COMP2["desktop.cordis.patch.yml<br>叠在 web-app 之上"]
    BUNDP --> COMP
    BUNDP --> COMP2
    COMP --> BOOT["boot<br>mountRootInclude + Loader"]
    COMP2 --> BOOT
    BOOT --> TREE[("Cordis 插件树")]
```

## 根因：怎么让"一切皆插件"既可一键跑起来，又允许零侵入定制

系统主张"无特权内核"，即运行时的组合（哪些服务、以什么配置装配）必须是数据而非代码。但要真能用，"组合"就得有落点：谁来写这份数据、写完怎么执行、怎么不污染共享代码。根因拆成三层矛盾统一：

- **一切皆插件与开箱即用的矛盾**：纯插件系统若要求用户先学会 Cordis patch 语法才能启动就不可用。部署层用"模板化 profile"化解：内置五套模板（`acp`/`web`/`headless`/`sdk`/`sdk-minimal`，各自带 bundle 列表与 `patchReload` 策略），首次使用自动初始化，让 `dsh web` 无需任何配置即能跑起完整 Web 形态（[packages/boot/app-boot/src/profile.ts:110-129]()
<!-- anchor: export const PROFILE_TEMPLATES: Record<string, ProfileTemplate> = { … patchReload: 'startup', -->）。
- **零侵入定制与共享 base 的矛盾**：用户改行为但不破坏"core 是同一份"的共享性。方案是分层 patch：base bundle 是所有 base 系 profile 的第一层、把整个 core 作为**一个 insert** 应用（[packages/bundle/base/cordis.patch.yml:15-17]()
<!-- anchor: - insert: … name: '@deepseek-ai/cordis-plugin-timer' -->），用户写在 profile 自己的 `cordis.patch.yml`、home 级 `$DSH_HOME/cordis.patch.yml`、`--patch` 覆盖层里——后层按 `id` 覆盖前层、**整块替换 `config`**而非深合并（`apps/cli/reference/README.md:9`）。"禁用而非删除"刻意保留半成品行，防止重排组合时静默复活（[packages/bundle/web-app/cordis.patch.yml:358-360]()
<!-- anchor: # Disabling rather than deleting is deliberate: the base is shared, and a row … # the composition. -->）。
- **安装插件与内核无特权的矛盾**：内核不该知道"npm 里的某个包是插件"。部署层用 manifest 契约翻译：bundle 是 npm 包，其 `dsh.bundle.patch` 声明 patch 文件；profile 是含 `dsh.profile.bundles` 顺序列表的目录（[packages/util/package-manifest/src/types.ts:26-29]()
<!-- anchor: /** The configuration layer exported by a bundle package. */ … patch: string -->）。于是"安装 = 加依赖，激活 = 打 patch"，插件的代码与配置行一起随 npm 包分发，内核只需消费契约而不知道 npm。

> **演进说明（0.1.5-rc.2）**：这两个 manifest 接口在 0.1.2 时定义在 `packages/boot/app-boot/src/profile.ts` 内（`DshBundleManifest`/`DshProfileManifest`），现已抽成独立包 `@deepseek-ai/dsh-package-manifest`，与 `client`/`configTrees`/`sessionFormatMigration` 等其它 `package.json.dsh` 角色共享同一个 `DshManifest` 聚合类型——契约从"launcher 私有"变成"全家共享"。

概括：**"部署即组成插件树"**——profile 命名组合、bundle 提供配置行+代码、patch 分层允许任意覆盖；内核只负责按层把 patch 应用成树，对"从哪来、谁来安装"保持无特权。

## 设计思路：双形态、三个实体、两种分发、一个诊断面

> **演进说明（0.1.5-rc.2）**：标题里的"双形态"指 0.1.2 时的 `web`/`headless` 两个随附模板；0.1.5 加入 Electron 桌面端后，分发矩阵实际是**三形态**。三者的差异依旧全部落在 patch 组合上，所以本节标题保留原文，新增 `### 桌面形态：Electron 复用同一套 bundle` 说明第三种发行如何复用既有分层。

### web vs headless 双形态：何时用哪个

两个随附模板 profile，路径都在 `$DSH_HOME/profiles/<name>`：

| 形态 | bundle 组合 | patchReload | 面向 | 进程生命周期 |
|---|---|---|---|---|
| `web` | base + web-app | `live`（用户 patch 热重载） | 长驻服务、浏览器 GUI | 常驻 http 服务，SIGINT/SIGTERM 优雅退出 |
| `headless` | base + headless | `startup` | 一次性任务 runner | 跑完打印结果即退出 |

- web 形态透传**宿主机行**与**前端（浏览器）插件 roster**：一个 `insert` 挂起 webserver、api-gateway、web-runtime、以及整套 `dsh-client-ui-*` 浏览器端插件行（[packages/bundle/web-app/cordis.patch.yml:44-177]()
<!-- anchor: - insert: … name: '@deepseek-ai/dsh-client-modules' -->）。默认 `http://127.0.0.1:3080`，端口来自 `ctx.webStartup.port ?? 3080`（[packages/bundle/web-app/cordis.patch.yml:135-140]()
<!-- anchor: - id: webserver … port: !!js ctx.webStartup.port ?? 3080 -->）。
- headless 形态**不挂 Host、HTTP、Web runtime、浏览器插件**：只插 `code-runtime` + `headless-startup` + `headless-runner` 三行（[packages/bundle/headless/cordis.patch.yml:18-31]()
<!-- anchor: - insert: … task: !!js ctx.headlessStartup.task -->）。运行逻辑：先 `await loader.await()` 等整棵树装完，再 `agents.create` 建 Agent → `agent.followup` 发任务 → `agent.whenIdle` 等静默 → `sessions.flush` 持久化 → `summarize` 取最后一条 assistant 文本 → `ctx.get('appExit')` 退出（[packages/bundle/headless/src/index.ts:169-211]()
<!-- anchor: async function run(ctx: Context, task: string, io: HeadlessIo): Promise<void> { … io.exit(outcome.reason?.kind === 'completed' ? 0 : 1) -->）。
- 何时用哪个不是架构强加，而是 bundle 组合的选择：`web` 想更轻不想要 HTTP，就在 profile 的 `cordis.patch.yml` disable webserver 行；headless 想要浏览器交互，就在 bundle 层加 web-app。双形态其实是"同一套 patch 组合谱系的两端样板"。

### 桌面形态：Electron 复用同一套 bundle，只换传输面

第三种发行是 `apps/desktop`（Electron 壳）+ `apps/desktop-host`（私有 Host 进程）。它对部署层提出的问题很尖锐：**要不要为桌面再造一个 bundle？** 答案是不——桌面端把 `web` 的组合原封不动装进一个独占 profile，再用一层 overlay 把这棵 Web 树里"服务端"的那半摘掉。

- **同一份 bundle 列表**：桌面 profile 的 `dsh.profile.bundles` 就是 `['@deepseek-ai/dsh-base', '@deepseek-ai/dsh-web-app']`，与 `web` 模板逐字相同（[apps/desktop/src/project-manager.ts:105-105]()
<!-- anchor: const DESKTOP_PROFILE_BUNDLES = ['@deepseek-ai/dsh-base', '@deepseek-ai/dsh-web- -->，同值也写进种子 manifest 与开发态 profile，见 [apps/desktop/src/project-manager.ts:694-694]()
<!-- anchor: dsh: { profile: { bundles: [...DESKTOP_PROFILE_BUNDLES] } }, -->）。区别只在 profile 目录（`$DSH_HOME/profiles/desktop`）由 Electron 独占，CLI 被门禁挡在外面（见上文 `rejectElectronProfile`），因此 CLI 与桌面可以共享 `$DSH_HOME` 下的会话/设置/凭据，却不共享可执行包、插件激活与 lockfile。
- **一层私有 overlay 换传输面**：Host 用 `loadProfileDirectory` 读同一个 profile 目录、把 bundle 层 + profile 层照常堆起来，然后**在最上面再叠一层自带 patch** 并把整摞交给 `boot`（[apps/desktop-host/src/index.ts:152-176]()
<!-- anchor: function desktopPatches(projectDir: string, allowLinkedPackages: boolean): Patch … return layers.flat() -->）。这层 patch 的正文只有二十来行：disable `web-startup`/`webserver`/`web-runtime`/`client-hmr`，disable `open-in-app`/`ui-open-in-app`/`directory-picker`，给 `connection` 行改 `inject`/`config`，再 insert 两个处理原生目录选择器的行（[apps/desktop-host/config/desktop.cordis.patch.yml:1-26]()
<!-- anchor: # Electron reuses the browser composition without its network and browser-launch … - credentials -->）。**"桌面 = web 组合 − 监听端口 + 原生替换件"被表达成数据，而不是代码分支**——这就是分层 patch 设计的直接红利：新增一种宿主环境不需要碰 `web-app` bundle 一行。
- **运行时也被钉住**：Electron 不用系统 Node/pnpm，而是在打包进应用的**上游 Node.js** 下 spawn 私有 Host 入口（`node_modules/@deepseek-ai/dsh-desktop-host/lib/index.js`），把 `NODE_OPTIONS`/`DSH_DESKTOP_*`/`npm_*`/`pnpm_*` 全部从子进程环境里滤掉（[apps/desktop/src/host-process.ts:99-118]()
<!-- anchor: async start(): Promise<DesktopHostReady> { … throw new Error('dsh desktop host did not expose the required byte pipes and IPC -->）。理由是 Electron 自带的 Node 带着 fuse/ABI/生命周期补丁，而系统运行时与包管理器状态不可控；把运行时钉死后，"壳 + dsh 种子 + Node + pnpm" 才构成一个**同版本、整体签名**的发行单元（[apps/desktop/src/release.ts:6-13]()
<!-- anchor: /** Release facts embedded in the seed and copied into the active desktop projec … readonly pnpmVersion: string -->）。
- **打包与更新走通用 provider**：electron-builder 配置把 mac 目标设为 `dmg`+`zip`（强制签名、hardened runtime、公证）、Windows 目标设为 `nsis`（强制代码签名），发布通道只有一个 `generic` provider URL（[apps/desktop/electron-builder.config.mjs:64-94]()
<!-- anchor: mac: { … target: ['nsis'], -->）。自动更新默认**不自动下载、不自动安装**，先由协调器取到可用版本再交给原生确认对话框（[apps/desktop/src/update-coordinator.ts:11-31]()
<!-- anchor: export class DesktopUpdateCoordinator { … this.updater.autoInstallOnAppQuit = false -->）。

```mermaid
graph TB
    subgraph SHELL["Electron 壳 apps/desktop"]
        MAIN["main.ts<br>单实例锁 + 窗口"]
        PROJ["project-manager.ts<br>staging 安装 + 回滚"]
        UPD["update-coordinator.ts<br>electron-updater"]
    end
    subgraph HOSTP["私有 Host apps/desktop-host"]
        HOST["index.ts<br>boot + dsh-app:// 资产"]
        OVER["desktop.cordis.patch.yml"]
    end
    subgraph LAYER["profile / bundle patch 层"]
        PROF["profiles/desktop<br>dsh.profile.bundles"]
        WEB["web-app patch<br>host 行 + 浏览器 roster"]
        BASE["base patch<br>共享 core"]
        NODE["上游 Node.js<br>+ 内置 pnpm"]
    end
    MAIN --> HOST
    PROJ --> PROF
    HOST --> OVER
    OVER --> WEB
    PROF --> WEB
    PROF --> BASE
    WEB --> NODE
    BASE --> NODE
    MAIN --> UPD
```

桌面的 launch 链路因此比 web 多一次"宿主换面"，其余阶段完全共用：

```mermaid
sequenceDiagram
    participant E as Electron main
    participant N as 上游 Node.js + pnpm
    participant H as dsh-desktop-host
    participant B as app-boot boot
    participant W as web-app patch

    E->>E: 单实例锁 + 恢复 activation journal
    E->>N: staging 内离线安装种子
    N-->>E: 写入 profiles/desktop
    E->>H: spawn(node, [入口, projectDir])
    H->>H: loadProfileDirectory(desktop)
    H->>B: boot(root, patches)
    B->>W: 挂 base + web-app 宿主行
    H->>H: 应用 overlay 关掉 webserver
    H-->>E: ready(protocol, dshVersion)
    E->>H: dsh-app:// fetch 走字节管道
```

> 桌面的签名、公证、种子分片、回滚与自动更新细节属另一个关注面，见 [desktop-app.md](desktop-app.md)。

### profile / bundle / patch：安装插件 = 加 bundle + 打 patch

三个实体层层负责：

1. **Profile（命名组合）**：`$DSH_HOME/profiles/<name>` 目录，含 package.json（依赖 + `dsh.profile.bundles` 顺序列表）与 `cordis.patch.yml`（用户层）。模块头注释是权威描述，点明 bundle 是 `dsh.bundle.patch` 声明的 npm 包、profile 是含该顺序列表的目录（[packages/boot/app-boot/src/profile.ts:5-23]()
<!-- anchor-base64:KiBBIHByb2ZpbGUgaXMgYSBkaXJlY3RvcnkgdW5kZXIgYCREU0hfSE9NRS9wcm9maWxlcy88bmFtZT5gIGhvbGRpbmcgYSDigKYgKiBAbW9kdWxlIEBkZWVwc2Vlay1haS9kc2gtYXBwLWJvb3QvcHJvZmlsZQ== -->）。
2. **Bundle（分发单元）**：npm 包，其 `dsh.bundle.patch` 声明 patch 文件（[packages/util/package-manifest/src/types.ts:26-29]()
<!-- anchor: /** The configuration layer exported by a bundle package. */ … patch: string -->）。`dsh.profile.bundles` 里按名字引用，顺序即应用顺序。
3. **Patch（行为切片）**：顶层 YAML 数组的 Loader 补丁条目，支持按 id 覆盖 config、disable、insert，复用 include 的 `entryListSchema` 用户方言（[packages/boot/app-boot/src/index.ts:220-225]()
<!-- anchor: // The include's YAML dialect (`!!js` scalars become expression nodes the … const userPatchesSchema = entryListSchema -->）；`parsePatchList` 强校验顶层数组与条目映射形态（[packages/boot/app-boot/src/index.ts:350-367]()
<!-- anchor: function parsePatchList( … return anchorInsertedPluginNames(parsed as PatchOptions[], file) -->）。

**0.1.5 新增：模板创建变成一条显式命令。** 此前"模板 profile 首次使用时自动初始化"只发生在 `loadProfile` 内部（缺目录就按 `PROFILE_TEMPLATES[name]` 建），用户无法中途改名、也无从复用模板；现在 `--from-default-profile <template>` 把同一件事提前成独立动作（[apps/cli/src/profile-boot.ts:104-152]()
<!-- anchor: export function initializeProfileFromDefault( … throw error -->）。它的设计要点在**拒绝语义**而非复制语义：

- 目标名不能是随附模板名，目标目录必须不存在——`mkdirSync` 独占创建，抢不到就按"已有 package.json / 仅有残留目录"分别报错，**绝不覆盖**已有 profile（[apps/cli/src/profile-boot.ts:119-138]()
<!-- anchor: if (Object.hasOwn(PROFILE_TEMPLATES, name)) { … `${NAME}: profile directory ${dir} already exists; choose an unused profile name`, -->）。
- 只复制模板的 bundle 列表与 `patchReload`，不读同名本地 profile 的依赖与 patch，也不持久化"继承自谁"——所以后续模板表变化不会倒灌已建 profile。
- 初始化提交在 bundle 解析与 boot 之前：后续失败时新 profile 留在盘上，重试时省略该选项即可。

**"安装插件 = 加 bundle + 打 patch"一体两面**：`dsh plugin add` 先把包装进 profile 目录，再按 `dsh.bundle` 声明自动把包名补进 `dsh.profile.bundles`（[apps/cli/src/plugin.ts:59-90]()
<!-- anchor: function reconcilePlugins(before: ProfileManifest, profileDir: string): void { … writeProfileManifest(profileDir, after) -->）；用户想改行为时只在 patch 层写几行 id 覆盖，无需改 package 或重装。dsh 不区分"依赖"与"插件"——有 `dsh.bundle` 的依赖就是插件，`update` 后新增该声明的包会自动激活。

### sdk-minimal：刻意不复用 base 的独立组合

`sdk-minimal` 是分发表上的异类：它**不叠在 `dsh-base` 之上**，自己那一份 insert 就是完整 Cordis 树（[packages/bundle/sdk-minimal/cordis.patch.yml:1-3]()
<!-- anchor: # Standalone minimal SDK application. Unlike the ordinary SDK profile, this … # User profile, home, and invocation patches still apply above it. -->）。这是有意的正交：base 是"全产品基础"，而 SDK 极简组合要的是**模型可见面最小化**，任何 `base` 行的加入都必须显式重述一遍，而不是被动继承。

> **演进说明（0.1.2-alpha）**：`sdk-minimal` 是随 0.1.2-alpha 新增的**刻意独立于 base bundle 的「双工具 SDK 编码 agent」极简组合**（`dsh --profile sdk-minimal`）：只带平台选择持久 shell + `str_replace_editor`、session 以未压缩 JSONL 持久化、模型由 SDK 初始化请求选择；**显式排除** base/Web/settings/managed credentials/telemetry/compaction/workspace instructions/skills/jobs/subagents，并以 danger-full-access 权限策略运行（[packages/bundle/sdk-minimal/README.md:2-3]()
<!-- anchor: description: "Standalone single-tool SDK profile for users who need a minimal cr … kind: "package-bundle" --> 的 standalone 自述；该行的工具计数已在下一版由 two-tool 改为 single-tool）。配套另新增 SDK application bundle（`profiles/sdk-app`）与 ACP application bundle（`profiles/acp-app`）两种预置 profile 组合。

> **演进说明（0.1.5-alpha.2）**：上述"双工具"已收窄为**单工具**。`feat(minimal): remove str_replace_editor from minimal profiles`、`feat(base): remove str_replace_editor from default tools`、`feat(sdk): disable str_replace_editor by default` 三个提交把编辑器从 `minimal` preset、`sdk-minimal` bundle 与 `dsh-base` 默认 roster 里同时摘掉：默认文件编辑改由 `read`/`write`/`edit` 承担，`str_replace_editor` 退为显式 opt-in 行（在 profile/home/`--patch` 里把 `tool-str-replace-editor` 设为 `disabled: false` 即可加回）。同一时期 `feat(minimal): align persistent bash output with the one-shot shell contract` 让持久 shell 的可见输出对齐一次性 shell 契约：每条命令都以 `[Command finished with exit code N]` 收尾（含 0），超时追 `[Command timed out or OOM]`，并把描述里的两句固定网络声明换成一句依赖环境的表述。理由写得很直白——这些极简组合服务的模型是照一次性 `bash` 训练出来的，任何表面差异都是白送的分布偏移。工具 roster 的完整语义见 [tools.md](tools.md)。

### npm 分发 vs 源码构建：两种运行同一份入口代码

`dsh` bin 包发布时只带 `lib/*.js` + `config`，装到 npm 后由 `node lib/bin.js` 直接跑（bin: `dsh` → `lib/bin.js`，`apps/cli/package.json`）。源码开发则用仓库根的 `dsh` script `node --import tsx/esm apps/cli/src/bin.ts`（根 `package.json` scripts）免构建直跑 TS。两者共用同一份 `parseDshArgs`/`runProfile` 逻辑，只差模块解析管线（`apps/cli/reference/README.md:110`）。

浏览器前端 dist 作为 `web-app` bundle 的普通依赖 `@deepseek-ai/dsh-web-frontend` 随包分发，所以部署态下 `npx @deepseek-ai/dsh web` 也能直接 serve 到 UI，不必在消费者侧跑构建；`resolveDistIndex` 从该包 exports resolve dist 入口（[packages/bundle/web-app/src/index.ts:162-165]()
<!-- anchor: function resolveDistIndex(): string { … return join(dirname(require.resolve('@deepseek-ai/dsh-web-frontend/package.json' -->）。桌面端复用了同一个前端包，但由 Host 自己的 `dsh-app://` 处理器按请求逐文件读取，并对 `distRoot` 做 resolve + realpath 两道越界校验（[apps/desktop-host/src/index.ts:208-212]()
<!-- anchor: const target = resolve(normalize(join(distRoot, pathname))) … if (realTarget !== distRoot && !realTarget.startsWith(distRoot + sep)) return new Response(null, { status: -->），不再经 HTTP server。

### Python SDK 的单文件运行时：分发单元下沉到打包面

部署层还有一条不走 npm 的发行路径：Python SDK 需要把一个完整 dsh 塞进单个可执行文件。这条链路的关键抉择出现在 0.1.5——**谁拥有"同一个可执行文件既是 dsh、又是沙箱 runner"的 dispatcher**。

打包后的运行时用私有环境变量 `DSH_SUBPROCESS_RUNNER` 选择"这次进程是 CLI 还是 native runner 引导器"（父进程侧定义见 [packages/subprocess/subprocess-local/src/runner-launch.ts:12-12]()
<!-- anchor: export const SUBPROCESS_RUNNER_ENV = 'DSH_SUBPROCESS_RUNNER' as const -->）。0.1.5 之前这个 dispatcher 位于 CLI 包内（`apps/cli/src/runtime-bootstrap.ts` → `lib/runtime-bootstrap.js`），于是"按环境变量分派"变成了**可发布 CLI 入口**的行为：打包运行时 spawn 出去的 node 命令只要走到这个入口，就先被那道判断接管。`fix(subprocess): move SEA bootstrap to runtime packaging` 把它搬成 Python 打包面自有的入口（[python/sdk-runtime/runtime-bootstrap.mjs:1-13]()
<!-- anchor: #!/usr/bin/env node … await runSelectedSubprocessRunner(selection) -->）：无选择器时动态 import `@deepseek-ai/dsh/lib/bin.js` 跑 `runCli()`；有选择器时**先从 `process.env` 删除该变量**再进 runner，嵌套进程因此不会再次命中 dispatcher。配套地，`@deepseek-ai/dsh-subprocess-local` 从 CLI 的运行时依赖降级为 devDependency（只留给 patch 层测试），改由运行时包声明为真依赖。

平台目标同时补齐 macOS x64：目标表从四个变五个（Linux x64/arm64、macOS arm64/x64、Windows x64），每项给出 wheel tag 与可执行文件名（[python/sdk-runtime/platforms.json:14-16]()
<!-- anchor: "macos-x64": { … "executable": "deepseek-harness-sdk-runtime-macos-x64" -->；发布矩阵的说明见 [python/development.md:82-82]()
<!-- anchor: The runtime distribution is wheel-only. The release pipeline publishes five plat -->）。运行时仍是**自包含**的：`dsh` console 命令只把它自己的参数转发给内置可执行文件，且强制要求显式 `DSH_HOME`，从不回落到 `~/.dsh`。

### `--dump-config` 诊断面：boot-free 看组合树

`--dump-config`/`--dump-default-config` 是部署层自身的可观测工具：不启动、不解析 app 参数，把 bundle 层、用户层、home 层逐一应用（[apps/cli/src/dump-config.ts:31-57]()
<!-- anchor: export function runDumpConfig( … process.stdout.write(renderConfigDump(NAME, join(loaded.dir, PROFILE_ROOT_FILENA -->），每个连续区间标注来源文件与覆盖它的 layer（[packages/boot/app-boot/src/index.ts:409-471]()
<!-- anchor: export function renderConfigDump( … return groupedDump(composed, provenance) --> 的 `renderConfigDump` 与 [packages/boot/app-boot/src/index.ts:475-484]()
<!-- anchor: function groupedDump( … lines.push(`# == ${currentLabel}`) --> 的 `groupedDump`），`!!js` 表达式保持不求值原样打印。`--dump-default-config` 专用于"用户层已坏但仍想看 bundle 原始组合"的恢复诊断——`userLayer:false` 跳过解析坏掉的 `cordis.patch.yml`（[packages/boot/app-boot/src/profile.ts:815-818]()
<!-- anchor: * @param options - `userLayer: false` skips reading `cordis.patch.yml`, so a … * @returns the loaded profile (empty `patches` when the user layer is skipped). --> 的参数注释）。0.1.5 起它同样接受 `--from-default-profile`：可以"先建目标 profile、再打印它的默认树、但不 boot"。

## 关键数据结构

### Profile manifest（profile 目录 package.json）

一句定位：声明一个 profile "由哪些 bundle 按什么顺序组合"的包级清单，`dsh.profile.bundles` 是该顺序列表，`dsh.profile.patchReload` 决定用户 patch 是热重载还是仅启动时读一次（[packages/util/package-manifest/src/types.ts:32-37]()
<!-- anchor: /** The bundle composition declared by a profile directory. */ … patchReload?: ProfilePatchReload --> 的 `DshProfileManifest`）。

`apps/cli/` 里 `composeProfile` 读取 `profile.layers`（每个 bundle 解析出的补丁层 + profile 自身 patchPath）来逐层组合，是这条数据结构的实际消费方（[packages/boot/app-boot/src/profile.ts:78-91]()
<!-- anchor: /** A loaded profile: resolved bundle layers plus the user's own patch layer. */ … patchReload: ProfilePatchReload --> 的 `Profile` 与 [packages/boot/app-boot/src/profile.ts:67-75]()
<!-- anchor: export interface ProfileLayer { … patches: PatchOptions[] --> 的 `ProfileLayer`；组装点在 [packages/boot/app-boot/src/profile.ts:789-797]()
<!-- anchor: const layers = bundles.map((packageName): ProfileLayer => { … return { packageName, packageDir, patchPath, patches: loadOverlayPatches(binName -->，消费点在 [apps/cli/src/profile-boot.ts:206-211]()
<!-- anchor: function allPatches(composed: ComposedProfile): PatchOptions[] { … ...composed.overlays, -->）。

### Bundle manifest（bundle 包 package.json）

一句定位：一个 npm 包如何自证"我是插件并携带配置行"，`dsh.bundle.patch` 指向相对其根目录的 patch 文件（[packages/util/package-manifest/src/types.ts:26-29]()
<!-- anchor: /** The configuration layer exported by a bundle package. */ … patch: string -->）。六个内置 bundle `dsh-base`/`dsh-web-app`/`dsh-headless`/`dsh-sdk-app`/`dsh-sdk-minimal`/`dsh-acp-app` 的 package.json 都声明同一形态；`loadProfileDirectory` 在应用前解析每个 bundle 的 patch 并用 `loadOverlayPatches` 加载，**声明缺失即 fail-loud**（不是"无 patch"），因为把一个没有 bundle 声明的包写进层列表属于配置错误（[packages/boot/app-boot/src/profile.ts:789-797]()
<!-- anchor: const layers = bundles.map((packageName): ProfileLayer => { … return { packageName, packageDir, patchPath, patches: loadOverlayPatches(binName -->）。

### Patch 格式（cordis.patch.yml）

一句定位：顶层 YAML 数组的 Loader 补丁条目——id 覆盖 config / disable / insert 列表，允许 `!!js` 表达式，配 include 的 `entryListSchema` 用户方言解析（[packages/boot/app-boot/src/index.ts:220-225]()
<!-- anchor: // The include's YAML dialect (`!!js` scalars become expression nodes the … const userPatchesSchema = entryListSchema --> 的 `userPatchesSchema = entryListSchema`；[packages/boot/app-boot/src/index.ts:350-367]()
<!-- anchor: function parsePatchList( … return anchorInsertedPluginNames(parsed as PatchOptions[], file) --> 的 `parsePatchList` 强校验顶层数组与条目映射形态）。

### patch 应用顺序（组合优先级）

`loadProfile` 把 bundle 层按 `dsh.profile.bundles` 顺序解析，profile 层在 bundle 层之后、home 层与 `--patch` 覆盖在最上（[packages/boot/app-boot/src/profile.ts:820-835]()
<!-- anchor: export function loadProfile( … return loadProfileDirectory(binName, dir, installAnchor, options) --> 的 `loadProfile`；缺 profile 目录时按模板 `initProfile`，见 [packages/boot/app-boot/src/profile.ts:825-832]()
<!-- anchor: if (!existsSync(join(dir, 'package.json'))) { … initProfile(dir, template.bundles, template.patchReload) -->）。`apps/cli` 侧由 `allPatches` 汇总成完整待应用序列（[apps/cli/src/profile-boot.ts:206-211]()
<!-- anchor: function allPatches(composed: ComposedProfile): PatchOptions[] { … ...composed.overlays, -->）。整块 config 替换而非深合并（`apps/cli/reference/README.md:9`）。桌面端在这条链的**最末端**再加一层自带 overlay，等于把"宿主差异"放进优先级最高的那一档。

## 核心流程：`dsh --profile web` 启动（CLI → loadProfile → 组合 → boot → web-app 挂 Browser UI）

```mermaid
sequenceDiagram
    participant U as 用户
    participant DSH as bin.ts
    participant ARGS as args.ts
    participant RUN as profile-boot.ts
    participant LOAD as app-boot loadProfile
    participant CORD as cordis Loader/boot
    participant WEB as web-app patch

    U->>DSH: dsh --profile web / dsh web
    DSH->>ARGS: parseDshArgs(argv)
    ARGS-->>DSH: mode profile, profile web
    DSH->>RUN: runProfile(profile web)
    RUN->>RUN: prepareProfile heal
    RUN->>LOAD: loadProfile web
    LOAD->>LOAD: 不存在则 initProfile 模板
    LOAD-->>RUN: layers base,web-app + patch
    RUN->>RUN: composeProfile 各层合成
    RUN->>CORD: boot(补丁, 先provide env)
    CORD->>CORD: mountRootInclude
    CORD->>WEB: 挂 base核心 + web-app宿主
    WEB-->>U: 打印 URL(http://127.0.0.1:3080)
```

流程要点（均带真实引用）：

1. **解析**：`bin.ts` 调用 `parseDshArgs`，`--profile web` 与 `web` 别名都收敛到同一 mode（[apps/cli/src/bin.ts:29-39]()
<!-- anchor: const invocation = parseDshArgs(process.argv.slice(2), readVersion()) … args: invocation.args, -->、[apps/cli/src/args.ts:126-150]()
<!-- anchor: export function parseDshArgs(argv: readonly string[], version: string): DshInvoc … .action((args: string[], options: BootOptions & { profile?: string }) => { -->）。
2. **prepareProfile**：`healProfilesModuleFallback` 重建 `$DSH_HOME/profiles/node_modules` flat symlink 回退目录，再 `loadProfile`（[apps/cli/src/profile-boot.ts:187-191]()
<!-- anchor: export function prepareProfile(name: string, userLayer = true, fromDefaultProfil … return profile -->）。
3. **loadProfile**：web 目录不存在则按模板 `initProfile` 为 base+web-app 两个 bundle（[packages/boot/app-boot/src/profile.ts:825-832]()
<!-- anchor: if (!existsSync(join(dir, 'package.json'))) { … initProfile(dir, template.bundles, template.patchReload) -->）；随后逐个 resolve bundle、加载其 patch layer、解析用户 patch（[packages/boot/app-boot/src/profile.ts:789-797]()
<!-- anchor: const layers = bundles.map((packageName): ProfileLayer => { … return { packageName, packageDir, patchPath, patches: loadOverlayPatches(binName -->）。
4. **composeProfile**：把 bundle 层、profile 层、home 层、`--patch` 覆盖、telemetry patch 合成应用序列（[apps/cli/src/profile-boot.ts:226-243]()
<!-- anchor: async function composeProfile( … return { profile, bundlePatches, homePatches, overlays: composedOverlays } -->）。
5. **boot**：造空 Context，装 Loader，`mountRootInclude` 挂根 include 并把补丁一次 `applyEntryPatches`（[packages/boot/app-boot/src/index.ts:787-804]()
<!-- anchor: export async function boot( … await mountRootInclude(ctx, absoluteConfigPath, patches, bareModuleBaseUrl) --> 的 `boot`；[packages/boot/app-boot/src/index.ts:516-558]()
<!-- anchor: export async function mountRootInclude( … return entry --> 的 `mountRootInclude`）。app 参数与环境以 `ctx.cmdlineArgs`/env 服务在树挂载前提供（[packages/boot/cmdline/src/index.ts:84-88]()
<!-- anchor: export function provideCmdline(ctx: Context, host: CmdlineHost): void { … if (host.ready !== undefined) ctx.provide('appReady', host.ready) --> 的 `provideCmdline`）。
6. **web-app 挂 Browser UI**：web-app patch 的 `insert` 挂 webserver（默认 3080）、api-gateway、web-runtime、以及整套 `dsh-client-*` 浏览器端 roster 行（[packages/bundle/web-app/cordis.patch.yml:44-177]()
<!-- anchor: - insert: … name: '@deepseek-ai/dsh-client-modules' -->）；`web-app` 运行时插件 resolve 前端 dist 并挂 `frontend-static` fallback、注册 web-surface prompt 与 `DSH_WEB_URL` bash 变量、打印 URL line（[packages/bundle/web-app/src/index.ts:225-248]()
<!-- anchor: export function apply(ctx: Context, config: Config): void { … resolve: () => ({ [DSH_WEB_URL]: localWebUrl(runtimeCtx) }), --> 的 `apply`）。浏览器端 `modules` 行 node 半区扫描树、浏览器半区作为 shell kernel 插件表构造（[packages/bundle/web-app/cordis.patch.yml:170-177]()
<!-- anchor: # ── browser plugin roster (dsh.client rows; node halves are layer-2 hosts) ── … name: '@deepseek-ai/dsh-client-modules' -->）。

### headless 的一次性流程（同部署层不同叶子）

`dsh --profile headless "run the tests"`：profile-boot 走同一组合，但 headless patch 只插三行（[packages/bundle/headless/cordis.patch.yml:18-31]()
<!-- anchor: - insert: … task: !!js ctx.headlessStartup.task -->），`headless-startup` 用 `parseCmdline` 把 task positional 发布为 service（[packages/bundle/headless/src/startup.ts:49-56]()
<!-- anchor: export function apply(ctx: Context): void { … parseCmdline(ctx, program) -->），`headless-runner` 作为该 service 的消费行跑完整轮并退出（[packages/bundle/headless/src/index.ts:169-211]()
<!-- anchor: async function run(ctx: Context, task: string, io: HeadlessIo): Promise<void> { … io.exit(outcome.reason?.kind === 'completed' ? 0 : 1) -->）。

### 桌面形态的启动链路（同部署层，另一层 overlay）

桌面端复用 `loadProfileDirectory` 而非 `loadProfile`——因为 profile 目录由应用自己的包工程拥有，不该再经 `$DSH_HOME` 解析一遍（[apps/desktop-host/src/index.ts:152-176]()
<!-- anchor: function desktopPatches(projectDir: string, allowLinkedPackages: boolean): Patch … return layers.flat() -->）。它额外做两件事：拒绝解析到 profile 目录之外的 bundle（除非开发态显式 `--allow-linked-profile`），以及把 `agent-presets` 行的 roots 重指到随版本分发的 `config/agent-presets`，让桌面会话的 preset 与壳同版本。其余 boot 阶段与 web 完全同一份 `boot`/`mountRootInclude`。

## 与其他模块协同

- **复用 app-boot 的 boot/profile**：本模块的启动引擎 100% 复用 `packages/boot` 的 `boot`/`loadProfile`/`composeEntries`，deployment 只是其上的薄 CLI 层——`profile-boot.ts` 的 import 一次命中全套（[apps/cli/src/profile-boot.ts:14-34]()
<!-- anchor: import { existsSync, mkdirSync, rmSync, writeFileSync } from 'node:fs' … } from '@deepseek-ai/dsh-app-boot' -->）。桌面端换的是**调用方**（Electron 壳 + 私有 Host）而不是引擎：`apps/desktop-host` 同样只 import app-boot 的 `boot`/`composeEntries`/`loadProfileDirectory`，自己不再实现一套组合逻辑。（跨模块边界：另见相应子文档）`boot` 具体挂载细节归 app-boot/boot 侧。
- **client/web 前端消费 `ctx.remote.<namespace>` + session/event**：0.1.2-alpha 起，web 形态浏览器端插件经 `dsh-client-connection`（node 半区把 gateway 绑到 webserver 的 `/api` 路由，浏览器半区是 fetch/SSE 客户端）与 `dsh-api-remotes`（显式挂载业务包的 `/remote` 贡献）把 host 面 Remote 控制器桥到浏览器——旧 apiproxy RPC（settings/credentials/directory-picker）已移除并由 Typert Remote 控制器取代（[packages/bundle/web-app/cordis.patch.yml:179-196]()
<!-- anchor: # Owns both ends of the web transport: node half binds the gateway to the … name: '@deepseek-ai/dsh-api-remotes' --> 的 connection 与 api-remotes 行）。（跨模块边界：Remote 体系的编程模型、生成管线与运行时调用见 [remote-controllers.md](remote-controllers.md)。）桌面端保留 `connection` 行但把它的 `inject` 改成只依赖 credentials，由 Host 自己的字节管道承载请求/响应。
- **Capability Seam**：部署层是 prompt/agent-preset 的落点——base patch 统一注入 system-prompt/agent-loop，web profile 暴露 agent-presets roster（[packages/bundle/base/cordis.patch.yml:465-487]()
<!-- anchor: - id: system-prompt … name: '@deepseek-ai/dsh-llm-deepseek' -->；[packages/bundle/web-app/cordis.patch.yml:44-484]()
<!-- anchor: - insert: … default: standard -->）。（跨模块边界：另见相应子文档）Seam 具体语义由 capability-seams 模块把关。0.1.5 起 base 的 persona 配置键从 `persona` 拆成 `personaPrefix`/`personaSuffix`，让"身份前缀"与"运行时上下文后缀"分别可换——headless 与 sdk-minimal 都靠它把 `{{cwd}}` 放到后缀位（[packages/bundle/headless/cordis.patch.yml:7-11]()
<!-- anchor: - id: system-prompt … You are a coding agent powered by the {{model}} model. -->）。
- **环境与凭据**：`loadLayeredEnv` 提供 frozen 环境快照，且禁止 `.env` 写 bootstrap 级变量（[packages/boot/app-boot/src/index.ts:92-137]()
<!-- anchor: /** Exact names no discovered file may set. */ … return BOOTSTRAP_NAMES.has(upper) || BOOTSTRAP_PREFIXES.some(prefix => upper.sta --> 的 `BOOTSTRAP_NAMES`/`BOOTSTRAP_PREFIXES`/`isBootstrapOnly`）；base 挂 `dsh-credentials-local` 行消费凭据（[packages/bundle/base/cordis.patch.yml:97-98]()
<!-- anchor: - id: credentials … name: '@deepseek-ai/dsh-credentials-local' -->）。0.1.5 给代理类变量开了唯一一个口子：`HTTP_PROXY`/`HTTPS_PROXY`/`ALL_PROXY`/`NO_PROXY` 可由 **Harness home 的 `.env`** 设置（调用目录的 `.env` 仍被拒），理由是代理决定每个请求的走向、而 home 文件是用户自己的（[packages/boot/app-boot/src/index.ts:92-137]()
<!-- anchor: /** Exact names no discovered file may set. */ … return BOOTSTRAP_NAMES.has(upper) || BOOTSTRAP_PREFIXES.some(prefix => upper.sta -->）。（跨模块边界：另见相应子文档）凭据解析链路属 credentials/data-model 侧。
- **Python SDK 运行时同样是部署产物**：`python/sdk-runtime` 是仓库的 pnpm workspace 成员，但它的分发形态是 **Python wheel + 单文件可执行文件**，不是 npm 包；其依赖闭包由 `scripts/build-exe-for-python-sdk.ts` 打包，入口 dispatcher 与平台表由该目录自有（[python/sdk-runtime/runtime-bootstrap.mjs:1-13]()
<!-- anchor: #!/usr/bin/env node … await runSelectedSubprocessRunner(selection) -->）。（跨模块边界：另见相应子文档）沙箱 runner 的语义归 sandbox/subprocess 侧。
- **Landlock 原生 addon 打包**：`native/landlock-run` 独立于本 workspace 但仍作为成员（根 `pnpm-workspace.yaml` 显式列 `native/landlock-run` 与其 `packages/*`）；提供 `landlock-run` 自限制后 exec 的沙箱二进制，按平台 npm optional 分发包，`probe` 探测不可用则调用方 fail-closed（`native/landlock-run/README.md` "Install"/"Usage" 段、[packages/bundle/base/cordis.patch.yml:214-252]()
<!-- anchor: - id: bash-sandbox … disabled: !!js process.platform !== 'win32' --> 的 bash-sandbox 行）。（跨模块边界：另见相应子文档）Landlock 与 sandbox 行在 Linux 上如何联同。

## 关键设计决策

### 决策一：为什么 web/headless 双形态而非单形态

- **Why**：headless 形态是"无服务器"的极简 runner——无 HTTP、API、浏览器，进程生命周期即一次任务的开始与结束，模块头直接声明 "no Host, HTTP runner, Web runtime, or browser plugin"（[packages/bundle/headless/cordis.patch.yml:1-5]()
<!-- anchor-base64:IyBUaGUgZHNoLWhlYWRsZXNzIGJ1bmRsZSBwYXRjaDogb25lLXNob3QgdGFzayBtb2RlIGRpcmVjdGx5IG92ZXIgZHNoLWJhc2UuIOKApiAjIGRyaXZlciBjcmVhdGVzIGFuIEFnZW50IHRocm91Z2ggdGhlIGNvcmUgcmVnaXN0cnkgYW5kIHByaW50cyBpdHMgZHVyYWJsZSByZXN1bHQu -->）。它输出一个即席 Agent 的最终文本并返回 0/1 退出码（[packages/bundle/headless/src/index.ts:207-211]()
<!-- anchor: io.stdout.write(outcome.text + '\n') … io.exit(outcome.reason?.kind === 'completed' ? 0 : 1) -->）。
- **权衡**：headless 不持久长驻、无交互授权面，适合无人值守批处理；web 形态提供 GUI/交互/权限提示，适合人工。二者不是两套独立实现，而是同一 base 上的**两种 overlay**——单形态语法上也表达得出这份差异，但将其固化为两个随附模板，把"我要哪种部署"变成 profile 层组合选择而不用写进内核。
- **连接哲学**：双形态正是"部署即组成插件树"的直观果实——同一内核既可长驻服务、又可跑完即退，行为差异完全由 patch 组合定义，内核零特权。

### 决策二：为什么 profile/bundle/patch 作为部署分发机制

- **Why**：把"软件能跑"拆成三个正交维度——**组合的命名**（profile）、**配置行+代码的分发**（bundle，随 npm 走）、**覆盖行为**（patch，用户侧零重装）。升级插件/换默认模型/改权限策略都退化为一个 patch 条目或一个 dependency 版本变化，无需触碰共享内核。
- **权衡**：代价是学习曲线与诊断复杂度——三实体须严格区分，否则"该改 profile 还是 bundle 还是 patch"困扰用户。项目用 `--dump-config`/`--dump-default-config` + reference 文档缓解；0.1.5 又把 profile 创建单列一条 `--from-default-profile`，正是把"新建一个 profile"从"猜它会不会自动出现"变成一条可读的命令。另一面收益是可靠的分发边界——bundle 解析先安装后 profile（[packages/boot/app-boot/src/profile.ts:751-758]()
<!-- anchor: export function resolveBundleDir( … throw new Error( --> 的 `resolveBundleDir`，先试安装锚点再试 profile 目录），保证内置 bundle 永远来自同一安装。
- **连接哲学**：无特权内核的必然推论——"怎么组合"必须是运行时可改写的数据，profile/bundle/patch 就是这份数据的标准化形态。

### 决策三：headless 无服务器一次性 runner 的取舍

- **Why**：一次性 runner 允许**丢弃型进程**——每次调用一个全新 Agent 与全新 Session，做完即死。无 HTTP、无持久绑定、无交互授权，天然适合 CI/无头/批处理。
- **取舍**：好处是零网络暴露、退出码语义简单、无状态可重复（每跑一次持久化一条 session）。代价是每次任务都要完整装配 core（Agent、模型选择、会话）后再跑（[packages/bundle/headless/src/index.ts:169-190]()
<!-- anchor: async function run(ctx: Context, task: string, io: HeadlessIo): Promise<void> { … installModelSelection(agentCtx, selected) --> 先 `loader.await()` 再 `agents.create`），冷启动成本高、无长驻缓存、无交互协商。
- **连接哲学**：headless 证明"内核可作库"——一次调用就是 `agents.create` + followup + whenIdle 的裸循环（[packages/bundle/headless/src/index.ts:184-201]()
<!-- anchor: const { agent } = await agents.create({ … await agent.whenIdle() -->），这正是 capability/事件抽象把"跑一个 Agent"做成普通 API 的回报。

### 决策四：桌面端为什么不新造一个 bundle

- **Why**：桌面端真正独特的只有两件事——**传输面**（不要监听端口，要走 Electron 的主进程管道/自定义协议）与**原生替换件**（目录选择器要用系统对话框）。其余（core、Agent、会话、模型适配器、前端 roster、agent-presets）与 web 形态逐字相同。为这点差异新写一个 `dsh-desktop-app` bundle，等于把 `web-app` 那份近 500 行的 patch（含整套浏览器 roster）复制一份并从此双份维护。
- **权衡**：代价是桌面端必须理解并覆盖一层别人的组合——overlay 里 disable 了 7 个 id（`webserver`/`web-runtime`/`web-startup`/`client-hmr`/`open-in-app`/`ui-open-in-app`/`directory-picker`），任何一次 `web-app` 行 id 改名都会打到桌面端；收益是**新增宿主环境的成本被压到一层 patch**，且"桌面到底跑了什么"仍可用 `--dump-config` 之类的手段按同一套层级读出。这是典型的"配置耦合换实现复用"，项目选了后者。
- **连接哲学**：与 base 被所有 profile 共享同构——**共享的是组合，不是代码**。桌面端证明 patch 分层不仅能做"用户微调"，也能承载一个完整发行形态的宿主差异。

### 决策五：SDK 运行时为什么把 dispatcher 收回到自己的打包面

- **Why**：一个单文件可执行文件要同时扮演"dsh CLI"和"沙箱 runner 引导器"，必须有个入口 dispatcher。这个 dispatcher 一旦长在**可发布的 CLI 包**里，"按 `DSH_SUBPROCESS_RUNNER` 分派"就不再只是 Python 打包面的私有行为——它是那个入口本身的语义，任何走到该入口的进程都要先过这道判断，包括 SDK 运行时 spawn 出去的 node 命令。
- **取舍**：搬到 Python 打包面后，CLI 包少了一个内部入口、多了一条"运行时依赖不得含 subprocess-local"的约束（改由 devDependency 承担测试职责）；换来的是 dispatcher 在进入 runner 前显式从环境删除选择器，**权责边界与环境清理都收敛到真正拥有它的那一层**。这是一次"把隐式全局行为改成显式局部行为"的重构，而不是功能变更。
- **连接哲学**：与 patch 分层同源——**谁产生差异，谁负责表达差异**；下游消费者不该替上游擦屁股。

## 全局关联：部署即「组成插件树」的外向面孔

三形态（web/headless/desktop）不是三套实现，而是同一 base 上的三种 overlay——长驻 GUI、一次性 runner、无端口桌面壳的差异全由 patch 组合定义，同一条 `boot` 让「内核可作库」（headless 的 `agents.create`+followup+whenIdle）、「内核可作服务」（web 的常驻 HTTP）与「内核可作桌面后端」（Host 的字节管道 + `dsh-app://`）同时成立。profile/bundle/patch 三个实体把「组合命名」「配置行+代码分发」「行为覆盖」正交拆开，升级/换默认/改权限都退化为一个 patch 或依赖版本变化；`--dump-config` 让「看到的 = 跑起来的」，`--from-default-profile` 让「新建一个组合」也成为一条可读的命令。分发边界同样是这条哲学的延伸：npm 侧的 bundle、Python 侧的单文件运行时、原生侧的 Landlock addon，各自只消费同一份 manifest 契约与 patch 方言。这正是 [architecture.md](architecture.md) 的「空树 + 分层 patch」哲学的落地——所有子文档里被分析的那些服务，最终都由这层数据化的组合决定是否、以什么形状出现在一次实际启动里。

> 部署相关但跨界较宽的实现（浏览器侧 `window.__DSH_BOOT__` 的注入/消费、浏览器 RPC 捆绑、桌面端的签名公证与种子分片、Landlock addon 三方包 manifest）其边界定义见 `packages/client`、`apps/desktop` 侧，本子文档聚焦部署形态本身。
