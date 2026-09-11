---
type: 项目解读
title: DeepSeek Harness 出站网络代理深读
description: dsh 0.1.5 的出站网络代理关注面（0.1.2-rc.1 起新增）——packages/util/http-proxy 把启动环境解析成一条进程级 ProxyPolicy 并装成 undici 全局 dispatcher，使 LLM adapter / web 搜索 / web_fetch / HTTP MCP / E2B 等 plain fetch 出站统一走代理；含 NO_PROXY 策略路由、loopback 结构性豁免、进程树与 Worker 继承缝、以及"模型自编代码不给代理"的逃逸防护。
tags: [deepseek-harness, network-proxy, undici, dispatcher, no-proxy, egress]
---

# 出站网络代理：一条进程级策略，覆盖进程树与 Worker 边界

> 子文档之一，聚焦 dsh 的**出站网络代理**关注面（`packages/util/http-proxy`，约 680 行源码 + 用户指南 `docs/user/guide/network-proxy.md`）。整体哲学见 [README](README.md)；本文与 [deployment.md](deployment.md)（profile 组合与启动顺序）、[subagent.md](subagent.md)（子进程/委派 CLI 的环境继承）、[webworker.md](webworker.md)（浏览器 Worker 里跑整棵插件树）在同一批链路上交汇。按 ai-agent ⊗ framework 类规格产出。

根因与一句定位：**Node 的内置 `fetch` 不认 `HTTP_PROXY`，而 harness 的出站调用点分散在 LLM adapter、web 搜索、`web_fetch`、HTTP MCP 与 E2B 里**——于是 dsh 不做"每个 provider 自己读环境变量"的横向铺开，而是把「这一进程的出站该走哪条路」收敛成**一个进程级答案**：启动时解析一次，装成 undici 的全局 dispatcher，让所有 plain `fetch()` 自动被覆盖。（遥测导出器是这条链路上**唯一被明确排除**的通道，理由见「权衡与不足」。）

## 根因：为什么必须由 harness 统一实现

这个问题不是"加个代理开关"，而是"谁来解释环境变量"。Node 的 `fetch` 会直接连出去，不管你导出了什么——[packages/util/http-proxy/src/index.ts:4-13]()
<!-- anchor: * Node's built-in `fetch` ignores `HTTP_PROXY` and friends, so every harness req … * Four functions, one per way a caller needs the policy — install it, ask how to -->
包注释第一句就点明了这条根因。用户在 shell 里 `export HTTPS_PROXY=...` 之后，`curl`/`git`/`npm` 都跟着走，唯独 harness 静默直连——这正是上游设计笔记记下的"实测"结论[.agents/notes/implemented/architecture/2026-08-27-outbound-proxy-policy.md:9-9]()
<!-- anchor: Node's built-in `fetch` ignores `HTTP_PROXY` and `HTTPS_PROXY`. Every other tool -->

把这件事下放给各 provider、或绕过统管另找开关，有四种被明确否决的做法：

| 备选方案 | 为什么不行（上游给出的判据） |
|---|---|
| 每个调用点自己读环境变量、自己建 transport | 调用点是**开放集合**：同类产品在 98 个点上逐个传策略值，第 99 个必然被遗忘；Pi 的 changelog 里 OAuth 与 Bedrock 就是两次事后补丁 |
| 只设 `NODE_USE_ENV_PROXY=1` 就收工 | 三条实测否掉：① 它在**进程启动时**采样环境，而 `loadLayeredEnv()` 之后才合并 `.env` 层，所以 `$DSH_HOME/.env` 里写的代理对它不可见；② 只有 Node 22.21+ / 24+ 认，而 `engines` 允许 `^22.19.0`；③ `web-fetch-http` 给 `fetch` 传自己的 `dispatcher`，**显式 dispatcher 覆盖全局**，标志再多也没用 |
| 用 `http.setGlobalProxyFromEnv()` | Node 自己的程序化开关，`fetch` 与 `node:http` 一起覆盖、还返回 restore 函数——但它 `added: v24.14.0`，22 线上完全没有 |
| 读操作系统的代理设置 | 六款同类产品里只有两款做且一款默认关；作者机器上实测 `scutil --proxy` 什么都没返回（代理软件写在 Wi-Fi 服务上，而主接口是 USB 网卡） |

这些取舍集中记录在仓库自带的设计笔记里，本文后续每条论断都能回到它或到源码。

## 概览：能力清单与技术栈

| 能力 | 说明 |
|------|------|
| 进程级一次安装 | 启动快照解析出一条 `ProxyPolicy`，`setGlobalDispatcher` 装上，所有 plain `fetch()` 免改代码被覆盖 |
| 策略路由 | `proxyForUrl()` 是唯一判据：scheme 选槽 + loopback 结构豁免 + `NO_PROXY` 后缀匹配 |
| 无 transport 的纯半边 | `policy.ts` 不 import `undici`，因此能在浏览器 Worker 运行时被求值 |
| 子进程继承 | `proxyEnvironmentForChild()` 叠加进 `scrubbedParentEnv()`，把用户原值还回去、把未命名的 scheme 补上解析值 |
| 逃逸防护 | 执行模型自编脚本的 workflow / code-runtime worker **完全不拿**代理配置（URL 可能带密码） |
| 拒值不致命 | SOCKS / 畸形 / 不支持 scheme 只报诊断并让该 scheme 直连，绝不阻止 agent 启动 |
| 回归门禁 | `verify-no-bare-dispatcher` 用 TS AST 拒绝自建 undici agent 与显式 `dispatcher`（带 `proxy-exempt:` 豁免） |

- **技术栈**：TypeScript + Node `engines ^22.19.0 || >=24.0.0` + `undici ^8.10.0`（peer 仅 `@deepseek-ai/cordis`）+ vitest；包身约 680 行（`policy.ts` 343 / `install.ts` 315 / `index.ts` 23）。

## API 面：从插件到 util 库，最终收敛到四个函数

这个包的形状被 review 打磨过两轮，值得单独记一笔，因为"导出什么"本身就是设计决策：

- **早期 revision 是一个 Cordis 插件**，导出膨胀到十七个，并且带一个没有任何 shipped bundle 挂载的 `Config` 分支；后来被整体拆掉，改成 `util/` 组里的**库**——理由是"transport policy has one answer per process"，没有比进程更窄的作用域可给，也就没有可 swap 的第二实现。
- 随后先收敛为 util 库的六个函数（`refactor(net): make the proxy a util library with six functions`），再在 `refactor(http-proxy): converge the proxy API on four functions` 中**收敛到四个**：`createNodeHttpAgent`（OTLP 专用）、`createDispatcher`（web_fetch 专用）、`proxyUrlFor`（E2B 专用）、policy accessor 各自被调用点改造吸收掉，而不是让包长出一个"SDK 约束目录"。

最终对外只有四个函数加一个类型，每个对应"调用方需要策略的一种方式"[packages/util/http-proxy/src/index.ts:17-23]()
<!-- anchor: clearedProxyEnv, … proxyRouteFor, -->

| 导出 | 职责 | 主要调用方与场景 |
|------|------|------------------|
| `installProxyFromEnvironment(env, report)` | 解析 + 报告诊断 + 安装，三者合成一步 | `apps/cli` 启动时唯一调用 |
| `proxyRouteFor(url)` | 回答单个 URL 走不走代理，**并且把该答案所假设的 transport 一起交出来** | `web-fetch-http`（分支决定是否 pin 地址）、E2B（SDK 自建 dispatcher） |
| `proxyEnvironmentForChild()` | 构造子进程需要的环境叠加层（`undefined` 表示删除） | `subprocess` 的 `scrubbedParentEnv()` |
| `clearedProxyEnv()` | 每个代理名都置 `undefined` 的叠加层，用于重放/隔离 | `loader-smoke`、`session-snapshot` 等测试夹具 |

包被有意切成"纯半边 + transport 半边"：`policy.ts` 不 import `undici`，因此浏览器 Worker 运行时照样能求值它[packages/util/http-proxy/src/policy.ts:2-7]()
<!-- anchor: * Proxy policy resolution: the pure, transport-free half of this package. It tur … * evaluates `dsh-web-fetch-http` without a Node transport. -->
而 `install.ts` 用**动态** `import('undici')` 把 transport 推迟到真正发请求时[packages/util/http-proxy/src/install.ts:6-6]()
<!-- anchor: * transport exists, matching how `dsh-web-fetch-http` defers its own transport i -->

`proxyRouteFor` 的形状是这轮收敛里最有信息量的一处：它不只回答"走不走"，proxied 分支**连带交出已经在这条策略上路由的进程级 dispatcher**[packages/util/http-proxy/src/install.ts:41-49]()
<!-- anchor: * How this process must send one request. … export type ProxyRoute = -->
因为"先读策略决定要不要 pin，再读一次策略去建 transport"之间存在卸载窗口——两次读之间发生 unmount，就会给一个"第一次读判定为走代理"的 URL 返回直连、未 pin 的 agent[packages/util/http-proxy/src/install.ts:62-66]()
<!-- anchor: export function proxyRouteFor(url: URL): ProxyRoute { … const proxy = proxyForUrl(policy, url) -->
把答案和 transport 绑成一个值，这个分歧类在构造上就不可能发生。

`installProxyFromEnvironment` 把解析、报错、安装合成一个操作，因为**没有调用方需要它们分开**——解析了不安装等于什么都没路由[packages/util/http-proxy/src/install.ts:296-301]()
<!-- anchor: export async function installProxyFromEnvironment( … for (const diagnostic of diagnostics) report(diagnostic.message) -->
唯一的调用点在 launcher，且在第一个插件挂载之前[apps/cli/src/profile-boot.ts:283-289]()
<!-- anchor: // Before the first plugin mounts and before anything can issue a request: Node' … (message) => { process.stderr.write(`${NAME}: ${message}\n`) }, -->
注意它读的是 **launcher 的环境快照**而不是 `process.env`——这正是"写在 `$DSH_HOME/.env` 里的代理也能生效"的原因，也是 `NODE_USE_ENV_PROXY` 无论怎样都做不到的那一点。

## 出站请求的完整链路

```mermaid
sequenceDiagram
    participant CLI as dsh 启动
    participant PX as util/http-proxy
    participant DS as 全局 dispatcher
    participant CALL as 出站调用方
    participant PXY as 用户代理
    CLI->>PX: installProxyFromEnvironment
    PX->>PX: 解析策略并收集诊断
    PX->>DS: setGlobalDispatcher 装 Agent
    PX-->>CLI: 返回 dispose
    Note over CALL: LLM adapter 与 web_fetch
    Note over CALL: 此后 plain fetch 均被覆盖
    CALL->>DS: fetch(url)
    DS->>PX: factory(origin) 求路由
    alt 该 URL 命中代理
        PX-->>DS: ProxyAgent(uri)
        DS->>PXY: CONNECT 或 absolute-form
        PXY-->>CALL: 响应
    else 本机 / NO_PROXY / 被拒 scheme
        PX-->>DS: undefined
        DS-->>CALL: 直连
    end
```

图里最关键的一步是 `DS->>PX: factory(origin) 求路由`：安装的 dispatcher 不是"自己再解析一遍环境变量"，而是**与 `proxyForUrl()` 共用同一个判据**，所以同一个 URL 不可能一处判定走代理、另一处直连。这也是为什么最终实现不能直接用 undici 现成的 `EnvHttpProxyAgent`——它在一个 scheme 没有对应代理时会**借用另一个 scheme 的代理**，恰好与本包"被拒的 scheme 保持直连"冲突[packages/util/http-proxy/src/install.ts:143-154]()
<!-- anchor: async function createPolicyDispatcher(policy: ProxyPolicy): Promise<Dispatcher>  … return new Pool(origin, passed) -->

调用点分三类，判断方式很干脆——**别猜 SDK 怎么发请求，去看它**：

- LLM adapter 与 `llm-pi-ai` 的发现请求都是 plain `fetch()`，装一次全局 dispatcher 就够。
- `web-fetch-http` 必须分支：走代理时跳过地址解析与 pinning（代理自己做 DNS），被 bypass 时仍走"解析 + pin"老路[packages/web/web-fetch-http/src/provider.ts:134-138]()
  <!-- anchor: const route = proxyRouteFor(url) … const addresses = await this.resolveAddresses(url.hostname, signal) -->
  代理腿交出的是进程级 dispatcher、调用方**不关闭**它[packages/web/web-fetch-http/src/network.ts:229-236]()
  <!-- anchor: export async function requestVia( … // proxy-exempt: the dispatcher is the installed policy's own, handed over by `p -->
  而直连腿自己 `new Agent(...)` 做地址 pinning，这属于"每请求状态、进程级 dispatcher 装不下"，因此带 `proxy-exempt:` 注释合法豁免[packages/web/web-fetch-http/src/network.ts:199-206]()
  <!-- anchor: const { Agent, fetch } = await import('undici') … const response = await fetch(url, { method: 'GET', redirect: 'manual', headers, -->
- E2B SDK 自建 dispatcher 且只认自己接到的 `proxy` 参数，所以它拿 `route.proxy`，判定还对**真正要调的 URL** 做[packages/e2b/e2b/src/index.ts:159-165]()
  <!-- anchor: const route = proxyRouteFor(new URL(e2bApiUrl())) … ...route.proxied ? { proxy: route.proxy } : {}, -->

补一句口径修正：**inbound webhook 不在这条链路上**。`packages/webhook` 是接收 `IncomingMessage` 的服务端规则运行时，自己不发 HTTP 请求，因此没有"webhook 走不走代理"的问题；真正需要被覆盖的出站方是模型调用、搜索、`web_fetch`、HTTP MCP 与 E2B。遥测导出器曾被纳入同一批改造，随后被 `revert(session-telemetry-otel): leave telemetry on its own transport` **刻意排除**——它是唯一"丢了也不损失任何功能"的出站通道（见「权衡与不足」）。

"装一次就全覆盖"这个承诺靠门禁守着：`verify-no-bare-dispatcher` 用 TypeScript AST（不是逐行正则，因为 `{ dispatcher }` 简写与重命名 import 都会漏）拒绝任何包自建 undici agent 或显式传 `dispatcher`[scripts/verify-no-bare-dispatcher.ts:2-13]()
<!-- anchor: * Verify that no package builds its own undici agent or hands `fetch` an explici … * pinning does — says so with the marker below. -->
门禁看不见依赖内部，所以每个出站调用点还配一个 `egress.spec.ts`，让真实代码路径穿过一个假代理、断言代理确实看到了请求——E2B 这样"读代码看走眼"的缺口就是这么被找出来的（它自建 Agent，最终改成接 `route.proxy`）。OTLP 导出器也被同一批测试照到（浏览器 transport 在 Node 上会换成 `node:http`，全局 dispatcher 够不到），但结论不是"接上"而是"保留直连"，于是那条 egress 测试**反向断言**：装上策略后假代理不应看到任何东西。

## 策略路由：环境变量如何变成一个 URL 的答案

### 环境名解析：小写优先、空白视同未设

`http_proxy`/`https_proxy`/`no_proxy` 三个字段各自拥有一对小写/大写名字，**永远成对写或成对清**[packages/util/http-proxy/src/policy.ts:39-42]()
<!-- anchor: export const POLICY_ENV_NAMES = { … noProxy: ['no_proxy', 'NO_PROXY'], -->
读取顺序是小写优先、大写兜底，并且**空白值视同未设**——后者不是洁癖：undici 自己的 `??` 链会让一个空的 `http_proxy` 遮住有值的 `HTTP_PROXY`，本包在解析阶段就把这个坑填掉[packages/util/http-proxy/src/policy.ts:109-117]()
<!-- anchor: function readEnv( … return undefined -->
除三对字段外，`ALL_PROXY`/`all_proxy` 也被解析（但**不**回写），`PROXY_ENV_NAMES` 是"所有携带代理配置的名字"的完整清单，供需要隔离子进程的调用方一次清干净[packages/util/http-proxy/src/policy.ts:50-52]()
<!-- anchor: export const PROXY_ENV_NAMES: readonly string[] = [ … 'all_proxy', -->

Windows 上这一对名字在 `process.env` 里是**同一个变量**。写入前必须先把全部名字快照下来，否则"读大写名字"会读回刚刚写进去的小写值，恢复时就把策略还回去而不是把用户环境还回去[packages/util/http-proxy/src/install.ts:114-121]()
<!-- anchor: // Snapshot EVERY name before writing any of them. Windows folds environment nam … else process.env[name] = value -->

### 回退链：`ALL_PROXY` 与 https→http

一个 scheme 的解析顺序是"自己的槽 → `ALL_PROXY` → （仅 https）http 代理"，与 undici 对齐；其中 `ALL_PROXY` 兜底是 Node 与 undici 都不自己做的[packages/util/http-proxy/src/policy.ts:306-324]()
<!-- anchor: export function resolveProxyPolicy(env: EnvLookup): ProxyResolution { … diagnostics, -->

### 被拒的值：该 scheme 保持直连，而不是借用别人的代理

用户显式命名了一个本包不支持的代理（`socks5://`、PAC、畸形串、非 http(s) scheme）时，处理是**报告并跳过**，且**该 scheme 保持直连**，不回退到 `ALL_PROXY` 或 http 代理[packages/util/http-proxy/src/policy.ts:194-196]()
<!-- anchor: if (own.kind === 'accepted') return own.value … return fallbacks.find(value => value !== undefined) -->
这条 `rejected` 与 `absent` 的区分是刻意的：用户为 https 指定了代理，若悄悄改成另一个 scheme 的代理，"诊断说保持直连、实际却 tunnel 走了别人的代理"，两者就矛盾了。诊断信息只报**变量名**、绝不回显值，所以 URL 里的用户名密码不会进入日志[packages/util/http-proxy/src/policy.ts:154-168]()
<!-- anchor: if (SOCKS_PROTOCOLS.has(parsed.protocol)) { … return { kind: 'rejected' } -->

### `NO_PROXY`：后缀语义 + CIDR 的刻意不支持

一条 bypass 条目命名一个 host，并**连同其所有子域**一起匹配（`example.com` 也 bypass `api.example.com`），前导 `.` 与 `*.` 同义，可带 `:port`，`*` bypass 一切[packages/util/http-proxy/src/policy.ts:279-294]()
<!-- anchor: export function bypassesProxy(noProxy: string, url: URL): boolean { … return false -->
用户文档把同一套语义写给用户，并明确 **CIDR 无效**：操作系统 bypass 列表里常见的 `10.0.0.0/8`、`192.168.0.0/16` 抄进 `NO_PROXY` 不会有任何效果，必须改写成域名后缀[docs/user/guide/network-proxy.md:34-42]()
<!-- anchor: `NO_PROXY` lists hosts to reach directly: … **CIDR ranges do not work.** An operating system bypass list often contains entr -->
"不支持 CIDR"是个不讨喜但诚实的取舍：真要支持就得自带一套 CIDR 匹配器，而 `NO_PROXY` 在 Node、curl、undici 之间的语义本就不统一——本包选择"明确说不支持"，而不是做出一个只有自己认的近似。

### 本机豁免：结构性识别，而不是四条字面量

loopback 永远直连，否则 Web UI、Connection transport 与所有本地测试服务器都会绕回代理形成路由环。发布出去的 bypass 列表里有四条字面量[packages/util/http-proxy/src/policy.ts:26-31]()
<!-- anchor: * Loopback entries merged into every policy's `noProxy`. A proxy that also serve … * matcher reads a bare `::1` as host `:` port `1` and therefore never bypasses i -->
`::1` 与 `[::1]` 都写，是因为这份字符串还要交给 undici——它的匹配器会把裸 `::1` 读成 host `:` port `1`，于是永远豁免不掉。

但只匹配这四条是不够的：`127.0.0.2`、整个 `127.0.0.0/8`、`0.0.0.0`、以及 IPv4-mapped 拼写都会漏过去。所以 `proxyForUrl` 这一侧**结构性识别本机地址**（含 `*.localhost`、`::`、`0.0.0.0`、`::ffff:127.0.0.1` 与 URL 归一化后的 `::ffff:7f00:1`），发布列表只服务"读环境变量、读不到策略对象"的消费者[packages/util/http-proxy/src/policy.ts:257-264]()
<!-- anchor: export function isLoopbackHost(hostname: string): boolean { … if (mappedHigh !== undefined) return Number.parseInt(mappedHigh, 16) -->
IPv4 octet 用正则限死 `0-255`，避免 `127.999.1.1` 被当成 loopback。

合并进最终列表时保留用户原始顺序与写法，只在缺少时追加 loopback 四条；`*` 已经 bypass 一切则原样返回[packages/util/http-proxy/src/policy.ts:210-210]()
<!-- anchor: return [...entries, ...LOOPBACK_NO_PROXY.filter(entry => !present.has(entry))].j -->

安全上的连带后果是：**配了代理就等于声明信任代理做目的地选择**。`web_fetch` 原本会在本地解析并校验公网地址、把连接 pin 到校验过的地址；走代理这条腿没有地址可 pin（代理自己做 DNS），于是跳过解析与 pinning。但**一个字面量私网地址不享受这个捷径**——它本来就不需要解析，跳过检查等于让"本机上的代理"够到 `169.254.169.254` 这类被检查拦掉的地址，所以它仍旧走被拒路径。

## 进程树与 Worker 边界：谁继承、谁不继承

```mermaid
graph TB
    MAIN["dsh 主进程：全局 dispatcher 已装<br>plain fetch 走代理"]
    SUB["子进程 bash / git / gh / MCP stdio<br>scrubbedParentEnv 叠加 overlay"]
    MAIN --> SUB
    SUB --> SUBOK["子 Node 22.21+ / 24+ 同路由<br>收到被拒值则扣下 flag"]
    MAIN --> WT["worker_threads workflow / code-runtime<br>workerSpawnEnv 只给 TMP / TEMP"]
    WT --> WTD["自有 globalThis 无 dispatcher<br>请求直连"]
    MAIN --> BW["浏览器 Worker preview<br>只求值 policy.ts 无 transport 半边"]
    BW --> BWD["worker 内 fetch 不经过策略"]
```

### 子进程：把用户原值还回去，再补上解析值

每个 spawner 共用的基座是 `scrubbedParentEnv()`：父环境去掉凭据形状的名字与所有 `DSH_*`，再叠加 `proxyEnvironmentForChild()`[packages/subprocess/subprocess/src/index.ts:64-75]()
<!-- anchor: export function scrubbedParentEnv(): Record<string, string> { … else env[name] = value -->
bash 工具与 shell 会话走的是同一个 `ctx.subprocess`（本地实现里 `childEnv()` 就建在它之上），因此自动继承同一份环境，无需各自处理[packages/subprocess/subprocess-local/src/spawn.ts:46-48]()
<!-- anchor: export function childEnv(extra?: Readonly<NodeJS.ProcessEnv>): NodeJS.ProcessEnv { … if (process.platform !== 'win32') return { ...env, ...extra } -->
这条叠加的规则是本包最讲究的一处：

- 用户**以任一大小写命名过**的 scheme，子进程拿到的是**用户原话**（含大小写），绝不是本进程归一化后的值——否则用户为 `curl` 设的 SOCKS 代理会被悄悄换成 HTTP 代理[packages/util/http-proxy/src/install.ts:229-248]()
  <!-- anchor: * The proxy environment a spawned child needs. … * read the variables themselves. -->
- 用户**两种大小写都没命名**的 scheme，则携带本进程的解析值。理由是路由一致性：`NODE_USE_ENV_PROXY` **不读 `ALL_PROXY`**，若只把用户原值交出去，一个从 `ALL_PROXY` 解析出代理的父进程会有一个直连的子进程。
- bypass 列表始终是**解析后**的那份（只会追加 loopback，不会丢用户写的东西），免得子进程把本机流量发给一个路由不了它的代理。

实现上先铺 `NODE_USE_ENV_PROXY: '1'`，逐字段决定用原值还是解析值，最后做一次自检[packages/util/http-proxy/src/install.ts:262-278]()
<!-- anchor: export function proxyEnvironmentForChild(): Readonly<Record<string, string | und … return overlay -->

### `NODE_USE_ENV_PROXY`：能覆盖的 Node 版本，与主动扣下 flag

子 Node 要认这套环境，得靠 `NODE_USE_ENV_PROXY`，它只到 Node 22.21+ / 24+，而 `engines` 允许 22.19 与 22.20——那两个版本上子 Node 保持直连。这是个**被文档承认的缝**：本进程内不依赖任何 Node 版本（每次请求都落到全局 dispatcher），只有跨界那一刻才分叉。

更微妙的是那个自检：**只要子进程会收到的 `HTTP_PROXY`/`HTTPS_PROXY` 中有任何一个是被本包拒绝过的值，就把 `NODE_USE_ENV_PROXY` 整个扣下来**。原因是实测的：Node 在该 flag 下会在运行程序之前解析这两个变量，遇到非 `http:`/`https:` 的 scheme 直接以 1 退出——一个为 `curl` 保留的 `socks4://` 会**让每个 Node 子进程在跑第一行之前就死掉**（MCP server、subagent CLI、`npm` 全中），而父进程此前只报告过"该 scheme 直连"。扣下 flag 后：子 Node 直连（与本进程对该 scheme 的报告一致），`curl` 仍读到为它保留的值。

依赖顺序上还有一处不那么显眼但必须正确的约定：**"用户原环境"这份记录属于最外层那次 install**。否则一个叠在 launcher 策略之上的内层安装，会把外层发布到 `process.env` 的归一化值当成"用户写的"记下来，之后每个子进程都继承一份用户从没要求过的归一化[packages/util/http-proxy/src/install.ts:79-90]()
<!-- anchor: function applyPolicyEnv(policy: ProxyPolicy): () => void { … inheritedProxyEnv = previousInherited -->

### Node worker thread：有自己 `globalThis`，就没有这条路

worker thread 有独立的 `globalThis`，也就有独立的全局 dispatcher——主线程装的那份**根本到不了它**[packages/util/http-proxy/src/install.ts:166-169]()
<!-- anchor: * A worker thread has its own `globalThis` and so its own dispatcher; installing … * credentials. A worker that needs the policy has to be handed one explicitly an -->
本仓库目前**没有任何 worker 装过它**，而且这是刻意的：两个 worker（workflow engine 与 code runtime）执行的都是**模型编写的脚本**，而代理 URL 可能带 `user:password`。于是 workflow 的 spawn 环境只给 Windows 需要的 `TMP`/`TEMP`（未构建时再加 `TSX_TSCONFIG_PATH`），其余一律不给[packages/workflow/workflow-worker-thread/src/host.ts:48-59]()
<!-- anchor: export function workerSpawnEnv( … return env -->
这就是本关注面的**逃逸防护**：模型自编代码要么自己配网，要么连不出去，绝不会从环境里读到操作者的代理凭据。`clearedProxyEnv()` 则服务于另一类需求——重放录制会话的 harness 必须能连到自己的 fixture 服务器，而不是开发机或 CI 导出的代理[packages/util/http-proxy/src/install.ts:306-314]()
<!-- anchor: * The environment overlay that removes every proxy name from a spawned child. … return Object.fromEntries(PROXY_ENV_NAMES.map(name => [name, undefined])) -->

### 浏览器 Worker：只求值没有 transport 的那一半

`webworker` preview 把整棵插件树塞进浏览器 Worker，其中会加载 `dsh-web-fetch-http`——它静态 import 本包的 `proxyRouteFor`。这条 import 之所以能成立，是因为 `policy.ts` 不 import `undici`（见上文纯半边设计），而 `install.ts` 的 undici 是动态 import，preview 只需**求值模块、不执行 transport 路径**就不会把 undici 拉进启动图。

这里有一段值得记的演进教训：本分支曾为"给 Node 核心 HTTP 模块发请求的 SDK"增加一个 `node:http`/`node:https` agent 工厂，浏览器 Worker 的 VFS packer 是**静态扫模块请求**的，于是该工厂让 preview 镜像打成不可解包，作者为此注册了一个 `node:https` 占位 stub（`mock/https.ts` + module-proxies 映射）。随后那个工厂本身在"收敛到四函数"中被删掉，没有调用方再 import `node:https`，stub 与其映射、测试一并回退。也就是说：**这次跨界的正确解法不是给 Worker 造能力，而是把不需要的能力从包里删掉**。

## 用户可见配置面

用户侧只有一条路：**环境变量**（或 `$DSH_HOME/.env`）。文档开头一句话概括了覆盖范围：模型调用、web 搜索、页面抓取、走 HTTP 的 MCP 服务器[docs/user/guide/network-proxy.md:5-5]()
<!-- anchor: DSH routes its outbound requests — model calls, web search, page fetches, and MC -->

| 项 | 现状 |
|---|---|
| 支持的代理 URL | 仅 `http://` 与 `https://`；凭据写在 URL 里（`http://user:pass@host:port`），诊断只回显变量名不回显值 |
| 不支持 | SOCKS（`socks5://` 等，报诊断并让该 scheme 直连）、PAC、操作系统代理设置 |
| scheme 回退 | 自己的槽 → `ALL_PROXY` → （仅 https）http 代理 |
| bypass | `NO_PROXY`，后缀匹配子域、可带端口、`*` 全 bypass；**CIDR 无效**；loopback 无需列出 |
| TLS 拦截型代理 | 需自行在启动前 `export NODE_EXTRA_CA_CERTS=...`（Node 只在进程启动时读） |
| 明确直连 | 本机地址；遥测导出器；模型自编代码的 workflow / code-runtime worker；`web_fetch` 指向字面量私网地址 |

### 从哪读：分层 `.env` 里的代理只在 Harness home 那一层被接受

配置来源是 `inherited 环境 → 调用目录 .env → Harness home .env` 的分层快照，其中代理名有一个**单点豁免**：项目自己的 `.env`（随 `git clone` 而来）仍然拒绝设置 `HTTP_PROXY` 等名字，只有 `$DSH_HOME/.env` 这一层接受，因为那是用户自己的文件，而 `DSH_HOME` 本身是 bootstrap-only，没有 `.env` 能把这个豁免挪到仓库控制的目录上[packages/boot/app-boot/src/index.ts:120-126]()
<!-- anchor: * The bootstrap names the Harness-home `.env` alone may set. A proxy chooses the … const HOME_LAYER_PROXY_NAMES = new Set(['HTTP_PROXY', 'HTTPS_PROXY', -->
拒绝时的报错还会给出双出口提示：可以 `export`，也可以写进 home 的 `.env`。

### 与 `ctx.settings` 的关系：没有关系，而且这是刻意的

代理**不经过 `ctx.settings`**，没有任何 settings 命名空间、也没有 cordis 配置分支。这不是遗漏，而是收敛的结果：早期的插件版本有一个 `Config`（含 `mode` 与 config 来源的字段），但没有任何 shipped bundle 挂载它，于是"配置面"是唯一不可达的分支——插件一删，解析就只剩环境这一个来源。当时给出的理由是规则性的：**环境是唯一来源，因此不存在一个该按 fail-loud 处理的配置面**；而一个拼错的代理变量不该阻止 agent 启动（它可能是为别的工具导出的）。

代价是代理与 harness 其他可热改配置**行为不一致**：改 `settings.yaml` 立即生效，改代理变量要重启进程。好处是"流量去哪"这件事不来自可热改状态，也不来自仓库里的文件。

## 权衡与不足

- **单点与延迟**：所有出站共享一条进程级 dispatcher（含连接池）。好处是连接复用、路由唯一；代价是代理成为全局单点，且多一跳 RTT。`dispose()` 用 `close()` 而不是 `destroy()`，让卸载时已在途的请求能收尾——代价是卸载并非瞬时。
- **"默认不代理本机"的安全取舍**：loopback 全段直连是防路由环的必需项，但它同时意味着**本机上的代理无法代理本机服务**，也意味着任何以 loopback 形式出现的内部端点不会经过代理的审计。`web_fetch` 对字面量私网地址的拒绝是补在这一侧的护栏，其余调用点没有等价的地址检查。
- **Worker 能力缺失即降级**：执行模型代码的 worker 完全不给代理，所以"模型自编脚本要走代理"这件事**做不到**（文档明说：只有脚本自己配置才能联网）。这是拿能力换凭据不外泄的定向取舍，不是遗漏。
- **跨 Node 版本的缝**：子进程依赖 `NODE_USE_ENV_PROXY`（22.21+/24+），且一旦环境里有被拒值就整体降级为直连；子进程还会用 **Node 自己的** `NO_PROXY` 规则匹配，与本包在分隔符与 IPv4 区间支持上不同——这个"第二个匹配器"是被文档承认而不是被消除的。
- **遥测直连**：OTLP 导出器走 `node:http`，全局 dispatcher 够不到；路由它要么要求 `http.Agent.proxyEnv`（22.21/24.5 才有，engines 范围内的旧版仍旧直连），要么换 SDK 的 `fetch` delegate（但那条路没有压缩，而 shipped bundle 开着 gzip）。权衡结论是"遥测是唯一丢了也用户无感的出站通道"，于是保留直连、并用 egress 测试把这条**排除**也断言下来（SDK 升级若悄悄改走 `fetch`，测试会红）。
- **pre-release 接口未稳定**：四个函数的形状是两轮 review 后的产物，而项目处于 developer preview。门禁 `verify-no-bare-dispatcher` 与每调用点的 `egress.spec.ts` 事实上是这套 API 的兼容契约——升级时它们比类型定义更能说明"哪些行为不能变"。

## 可借鉴的设计

- **把"答案"和"该答案所假设的 transport"绑成一个值**。`ProxyRoute` 用一个联合类型消灭了"两次读策略之间的窗口"这类竞态，比加锁或加版本号都便宜，可迁移到任何"先判断、后按判断取资源"的场景。
- **发布面与判定面分离**：给"读环境变量的消费者"发四条字面量，给"读策略对象的内部消费者"用结构性判定（整段 loopback 识别）。同一语义在两个能力不同的出口上各给一份**能表达**的形式，而不是把弱的那个当唯一真相。
- **安全边界靠"不给"而不是靠"过滤"**：模型自编代码的 worker 不是"过滤掉凭据"，而是**根本不给代理配置**，并让每个出站调用点配一条 egress 断言把"不该走代理的确实没走"测出来——把一条**排除**也变成可失败的测试，是这套防护里最值得抄的一点。
- **拒值只报告、不阻塞**：环境变量可能是为别的工具导出的，"本包用不了"不等于"用户配错了"，因此报诊断 + 让该 scheme 直连 + 拒绝回退到别的 scheme，三件事同时成立才自洽。

## 收束

出站代理看起来是"读三个环境变量"，实际暴露的是 dsh 的一贯设计张力：**上下文（这里是一个进程的出站路由）必须收在能全局唯一裁决的地方**，凡是"每个插件自己办"的地方都会长出第 N 个漏掉的调用点。它也因此成为"库而非插件"的一个标准样本——`util/http-proxy` 不能挂载、不能换实现、没有比进程更窄的作用域，却仍是 `apps/cli`、`subagent`、`workflow`、`web`、`e2b` 五个组共同依赖的一层；而它与 [subagent.md](subagent.md)、[webworker.md](webworker.md) 的交界处，正是"继承"与"不给"这两种相反策略在同一份策略对象上的分岔。
