# DeepSeek Harness 核心提示词：插件化装配注册表

> 本解读针对 `/Users/morehao/Documents/study/ai-agent/deepseek-harness` 中 `packages/core/system-prompt` 及下游各工具的提示词 section。核心提示词**不是一条写死的文本**，而是一个「每次模型调用前动态拼装」的插件化注册表。

## 一句话根因

DeepSeek Harness 的系统提示词 = **`dsh-system-prompt` 注册表**（每个插件贡献一个「段」，按 `order` 排好、插值 `{{变量}}`、附上工具 schema），在**每次模型请求前**由主循环调用 `assemble()` 拼成完整提示词。这让「模型被告知了它是什么（身份）」与「它知道自己能做什么（工具）」都被降格成可插拔的注册物——与 README 的「无特权核心」根因完全一致。

## 三段式组装模型

完整系统提示词由三类 `PromptSection` 拼接（`packages/core/system-prompt/src/index.ts:52-75`），靠 `order` 约定分区：

| 约定 | order | 拥有者 | 内容 |
|---|---|---|---|
| **身份** identity | `-100` | 注册表自身 | 固定开场 `You are an AI agent powered by DeepSeek Harness.` |
| **人设** persona | `0` | 部署配置/agent | 部署人格模板，`{{变量}}` 可插值 |
| **工具指导** tool guidance | `100–199` | 各工具插件 | 跨调用的最佳实践，如 `tool:bash`、`tool:read` |

这样分区让「身份/人设」与「每个工具自己的行为指导」各归其主，互不污染。

### 身份：`harness:identity`

注册表构造时注入，order `-100`，默认开启（`index.ts:357-363`）：

```ts
this.section({
  name: 'harness:identity',
  order: -100,
  text: 'You are an AI agent powered by DeepSeek Harness.',
})
```

可用 `config.includeHarnessIdentity: false` 关闭（`index.ts:340`），仅当某个兼容性部署想自己拥有一整条系统提示词时才需要。

### 人设：`deployment:persona`

order `0`，文本来自 `config.persona`（`index.ts:364-369`），调用 `renderPrompt` 时若为空则整段被丢弃（`index.ts:213`）。常量名与顺序被导出（`index.ts:128-131`），因为 agent 作用域可以**同名遮蔽**它（见下）。

### 工具指导段：`tool:N`（order 100–199）

每个工具插件在注册工具 schema 的同时，贡献少量「跨调用指导」文本，而非把长文案塞进单次调用的 schema prose。实测锚点：

- `tool:read`（order 100）—「用 read 而非 `cat` 检查文本文件，结果带行号，用 offset/limit 继续读大文件」`packages/fs/tool-fs/src/read.ts:70-74`
- `tool:edit`（order 102）—「用于文本文件定向替换，默认 old_string 必须唯一，若重复需更具体或用 replace_all；本会话内需先 read」`packages/fs/tool-fs/src/edit.ts:77-81`
- `tool:bash`（order 105）—「检查每次 bash 结果的 `[exit code: N]` 标记，失败先调查再继续」`packages/shell/tool-bash/src/index.ts:236-240`

更多：`tool:pwsh`、`tool:write`、`tool:glob`、`tool:grep`、`tool:web_search`、`tool:web_fetch`、`tool:pty`、`tool:lsp`、`tool:cordis`、`tool:session-query`、`tool:ralph`、`tool:jobs`、`tool:report`、`tool:goal` 等，各自在对应 `packages/*/tool-*/src` 中注册，order 覆满 100–112+。

## 装配流水线 `assemble()`

`assemble(context)` 是每次模型步骤的入口（`index.ts:467-542`），顺序：

1. **解析变量**：全局变量 + 作用域层链变量（最近的 scope 胜出同类名，`index.ts:473-482`）。
2. **合并 section/context**：全局与作用域按名字合并，作用域遮蔽全局（`index.ts:484-485`）。
3. **收集工具 schema 并排序**：每个 `tools` provider 求值，`structuredClone` 摘除参数，按 `toolOrder` 配置或字典序规范化排序（`index.ts:487-503`, `164-178`）。
4. **抛给 waterfall**：`system-prompt/assemble` 瀑布让专家监听器协作改/替换整个装配（scope 过滤，`index.ts:532-535`）。
5. **施加 complete 段与 context 抑制**（`index.ts:536-541`）。

`orderTools` 支持 `toolOrder` 配置式排序（`index.ts:139-140` 的 `<unlisted-tools>` 占位、`146-157` 校验），把「模型看到的工具顺序」也做成显式配置，替代隐式的「按注册顺序」。

### 渲染：`renderPrompt()`

（`index.ts:212-217`）：每段进行**严格插值** `{{variable}}`（`index.ts:258-295`）→ 丢弃空段 → 用空行 `\n\n` 拼接。严格性指 fail-loud：引用未知变量（含 `{{constructor}}` 这类 prototype 名，用 `Object.hasOwn` 排除，`index.ts:283-286`）、已注册但无值、畸形完整 `{{…}}` 组、以及「有 `{{` 而没有闭合 `}}` 却仍有后续 `}}`」都会抛错，绝不发出脏提示词。单边未匹配的 `{{` 会被当作字面行文保留，替换值不再被二次扫描（`index.ts:287-294`）。

## 变量系统：`{{provider}} / {{model}} / {{cwd}}`

插件用 `ctx.systemPrompt.variable(name, provider)` 注册变量，主循环 `agent-loop` 预置三个（`packages/core/agent-loop/src/index.ts:351-353`）：

```ts
ctx.systemPrompt.variable('provider', context => context.agent?.options.provider)
ctx.systemPrompt.variable('model',    context => context.agent?.options.model)
ctx.systemPrompt.variable('cwd',      context => context.agent?.session.header.cwd)
```

变量在 section 文本中以 `{{name}}` 引用，渲染时按引用者上下文求值。命名约束 `[a-z][a-z0-9_]*`（`index.ts:134`），同名重复注册、非法名直接抛错（`index.ts:447-454`）。这正是"部署人设可写 `你是 {{model}} 上的编码 agent，工作目录 {{cwd}}`"这类模板的基础。

## 作用域遮蔽：同名即替换

核心机制：**agent 作用域注册同名 section 会遮蔽全局**，而不是叠加（`index.ts:483-485`）。`deployment:persona` 是这一模式的代表用例：

- `preset/persona` 插件在 agent scope 内注册 `config.text` 为 persona（`packages/preset/persona/src/index.ts:33-52, 55-57`），并断言「在 agent scope 之外挂载」会与注册表自身注册相撞而 fail-loud（README）。
- 子 agent 委派时也会在 child 作用域注册 `deployment:persona` 遮蔽部署人设（`packages/subagent/subagent/src/child-agent.ts:172`），实现"每个子 agent 有自己的人格"。
- 同名替换的代价是：注册表自身必须保留 `PERSONA_SECTION` 这个名字可被遮蔽（`index.ts:122-128` 注释明确点出）。

### 完整提示词与动态上下文

- **complete 段**：某 section 设 `complete: true` 后，waterfall 照跑（还能解析工具/变量），但结束后该段被恢复为**唯一** prompt section（`index.ts:505-517`）。多于一个 effective complete 段会让 assemble 失败（`index.ts:506-508`）。`preset/persona` 的 `complete` 配置即走此路。
- **动态 context 快照**：`PromptContext`（order 有序）是独立的 user 角色快照，与 system-prompt 段分开。`includeRuntimeContext: false` 或 `suppressRuntimeContext()`（`index.ts:415-421`）可整体关掉，便于"我只要自己的一条身份提示词"。

## 关联文档

- **token / KV-cache 效应**：身份是每请求固定成本；段文本每请求重复，随渲染内容缩放。前缀稳定性决定 KV-cache 复用——身份/人设/变量/段文本/顺序任一变化都可能从第一个变化的 token 起失效。与 [`context-and-compaction.md`](context-and-compaction.md) 的 token 计量打通。
- **工具 schema**：schema 与段是**相互独立的装配输入**，schema 随 schema 重复、段是 `tool:` 段文本。限制一个工具不会顺手移除它独立的指导段，反之亦然（见 [`tool-calling.md`](tool-calling.md)）。
- **Model-visible means logged**：拼装出的提示词与工具 schema 会被记录到会话日志，可被 `tokenMeter` 为压缩压力回放——正是 [`session.md`](session.md) 那条"模型所见必被记录"的不变量在此处的落地。

## 可借鉴的设计

1. **把"模型身份/人设/工具指导"拆成可插拔、按 order 排序的段**：各插件只贡献自己拥有的那句,不必拥有一整条系统提示词。
2. **作用于同名遮蔽而非叠加**：全局人设可被 agent 作用域精确替换，天然支持"每个子代理不同人格"。
3. **fail-loud 的严格 `{{变量}}` 插值**：对未知/无值/畸形引用直接抛错，宁可拒绝也不发出脏提示词。
4. **工具顺序做成显式配置**（`toolOrder` + `<unlisted-tools>` 占位），把"模型感知的工具排序"也视为可声明的配置而非副作用。
5. **complete 段作为逃生舱**：想要"一整条自拥提示词"时，一个 `complete: true` 段即可覆盖其他所有段，同时保留工具/变量的解析能力。
