---
type: 迁移指南（面向使用者的 V1 → V2 升级说明）
title: "OpenCode V1 → V2 迁移指南"
description: 面向使用者的升级指南：V1 与 V2 在 CLI 包名、配置 schema、插件 API 上的差异怎么迁。覆盖自动归一化的覆盖面与边界（顶层字段改名、provider 的 npm→package、options→settings、api→baseURL、modalities/tool_call→capabilities、permission/tools→permissions 及动作改名、provider id 改名）、迁移后最常见的 package 缺失报错及其成因、终端配置 tui.json+kv.json→cli.json 的一次性搬迁、指令文件只认 AGENTS.md，以及验证与回滚。
tags: [opencode, migration, v1-to-v2, cli, provider-config, config-schema, rollback]
---

# OpenCode V1 → V2 迁移指南

> **基线**：本文所有源码引用锚定 `v2.0.11`（`9eb6902aaf`）。引用路径一律相对仓库根，行号只在该 tag 的工作树下有效。
>
> **与深度解读的分工**：本文回答"**怎么把实例迁到 V2**"，是操作指南；"为什么这么设计"见 [`llm-route-system.md`](llm-route-system.md)（package 与路由装配）、[`skill-system.md`](skill-system.md)（配置分层发现与归一化）、[`data-model.md`](data-model.md)（v1→v2 持久层迁移）、[`deployment.md`](deployment.md)（上游安装器与分发形态）。

## 一、先估算迁移规模

V2 换了三样东西，工作量的上限由"你用到了几层"决定：

| 层 | V1 | V2 | 迁移方式 |
|---|---|---|---|
| **CLI 包** | `opencode-ai` | `@opencode/cli` | 手工换包 |
| **配置** | `provider` / `plugin` / `options` / `modalities` | `providers` / `plugins` / `settings` / `capabilities` | **自动归一化**，多数不用改 |
| **插件** | V1 插件 API | V2 插件 API | 需插件侧提供 V2 版本 |
| **持久层** | v1 数据库 | v2 数据库 | 上游一次性迁移（见 [`data-model.md`](data-model.md)） |

包名的权威来源就在包里——`@opencode/cli`，版本 `2.0.11` [packages/cli/package.json:3-4]()
<!-- anchor: "name": "@opencode/cli", … "version": "2.0.11", -->；它同时注册 `opencode` 与 `opencode2` 两个 bin [packages/cli/package.json:8-9]()
<!-- anchor: "opencode": "./bin/opencode.cjs", … "opencode2": "./bin/opencode2.cjs" -->。

**V1 与 V2 不能并存。** 两者都向全局 bin 目录写同名可执行文件 `opencode`（V2 的 bin 声明见上引），因此包管理器层面会相互覆盖或冲突；换包前先卸载 V1，否则新装会因目标已存在而失败。

## 二、配置：先看自动归一化的覆盖面

V2 读配置时先过一层 V1 归一化，因此**大部分 V1 配置不需要手改**。归一化的入口在 `config/normalize.ts`，它把旧词表逐项折成新结构；以 `provider` / `providers` 为例，旧字段先解码成 V1 类型、再交给 `ConfigMigrateV1.migrateProvider`，最后与新字段的声明合并 [packages/core/src/config/normalize.ts:168-177]()
<!-- anchor: const legacyProviders = migrateProviders(input.provider, diagnostics) … ) -->。

顶层字段的改名清单（均可自动完成）：

| V1 | V2 |
|---|---|
| `plugin`（数组） | `plugins` |
| `provider` | `providers` |
| `autoupdate` | `update`（`false`→`disable` / `"notify"`→`notify` / `true`→`auto`） |
| `permission` + `tools` | `permissions`（动作与资源两级规则合并） |
| `attachment` | `media` |
| `command` / `reference` / `skills` | `commands` / `references` / `skills` |

上表整张来自同一个顶层迁移函数的返回对象——它逐项把旧字段写进新结构 [packages/core/src/v1/config/migrate.ts:44-65]()
<!-- anchor: formatter: info.formatter, … providers: providers(info.provider), -->；`autoupdate` 的三值映射与 `permission`+`tools` 的合并分别在它之前单独处理 [packages/core/src/v1/config/migrate.ts:31-36]()
<!-- anchor: ? "disable" … : undefined, --> [packages/core/src/v1/config/migrate.ts:40-40]()
<!-- anchor: permissions: permissions(info.permission, info.tools), -->。

权限动作也被改名，否则迁移后的规则会匹配不到新工具名：`write` / `patch` → `edit`、`task` → `subagent`、`bash` → `shell` [packages/core/src/v1/config/migrate.ts:117-122]()
<!-- anchor: export function normalizeAction(action: string) { … } -->。

### provider 条目内部的字段映射

provider 是最容易迁错的一层，因为它同时涉及字段改名和"运行时包从哪来"：

- **`options` → `settings`**，且 `options.headers` / `options.body` 被**剥出来**成为顶层 `headers` / `body`，不再留在 settings 里 [packages/core/src/v1/config/provider-options.ts:11-24]()
<!-- anchor: export function provider(options: Options): ProviderResult { … headers: headerOverlay, -->；映射后的完整形状见 [packages/core/src/v1/config/migrate.ts:247-253]()
<!-- anchor: function migrateStandardProvider(info: ConfigProviderV1.Info) { … settings: info.api ? { ...options.settings, baseURL: info.api } : info.options ? -->。
- **`api` → `settings.baseURL`**（当两者都存在时 `api` 覆盖 settings 里的 `baseURL`）[packages/core/src/v1/config/migrate.ts:247-253]()
<!-- anchor: function migrateStandardProvider(info: ConfigProviderV1.Info) { … settings: info.api ? { ...options.settings, baseURL: info.api } : info.options ? -->。
- **`npm` → `package`**，这是关键的一条，下一节单独展开。
- **model 的 `modalities` / `tool_call` → `capabilities`**：只有当 V1 显式写了 `tool_call`、`modalities.input`、`modalities.output` 中任一项时才产出 `capabilities`，其中未写的子字段回落到 `Model.Capabilities.default()`；三项全空则整个 `capabilities` 不产出，交给下游默认值兜底 [packages/core/src/v1/config/migrate.ts:323-331]()
<!-- anchor: const defaults = Model.Capabilities.default() … : undefined -->。
- **provider id 本身会被规范化**：`azure-cognitive-services` → `azure`、`google-vertex-anthropic` → `google-vertex` [packages/core/src/v1/config/migrate.ts:298-302]()
<!-- anchor: export function providerID(input: string) { … } -->。

### 归一化的两条硬边界

这两条决定了"为什么我明明写了却像没写"：

1. **未知字段被静默丢弃。** 解码器带 `onExcessProperty: "ignore"` [packages/core/src/config/normalize.ts:45-45]()
<!-- anchor: const options = { errors: "all", onExcessProperty: "ignore", propertyOrder: "ori -->，所以往 V1 语法的条目里塞 V2 字段（例如把 `package` 写进 `provider.<id>`），该字段既不报错也不生效——V1 解码器眼里它不存在。
2. **原生字段赢。** 新旧词表同时出现时，先放旧值、再用新值覆盖，两者不等时记一条 `conflict` 诊断但**保留原生值** [packages/core/src/config/normalize.ts:731-742]()
<!-- anchor: function mergeMaps( … return result -->。想确认最终结果，用 `opencode debug config` 看归一化之后的文档，而不是猜。

## 三、最常见的迁移故障：`Unsupported package`

换到 V2 后最典型的报错是：

```
Unsupported package for <provider>/<model>:
```

它的来源是 `ModelResolver` 在解析模型运行时抛出的 `UnsupportedPackageError`，消息模板就是这一行 [packages/core/src/model-resolver.ts:28-38]()
<!-- anchor: export class UnsupportedPackageError extends Schema.TaggedError<UnsupportedPacka … } -->。触发条件是**解析不出可用的运行时包标识**：即 `Provider.packageName(model.package)` 返回空 [packages/core/src/model-resolver.ts:217-218]()
<!-- anchor: const specifier = Provider.packageName(resolved.package) … if (!specifier) return yield* unsupported(resolved) -->。

**根因**：V2 只为自定义 provider 迁移 `package`，而它的**唯一来源是 V1 的 `npm` 字段**——V1 provider 结构里 `npm` 是可选的 [packages/core/src/v1/config/provider.ts:77-84]()
<!-- anchor: export const Info = Schema.Struct({ … blacklist: Schema.optional(Schema.mutable(Schema.Array(Schema.String))), -->，迁移逻辑写成 `package: info.npm ? Provider.aisdk(info.npm) : undefined` [packages/core/src/v1/config/migrate.ts:247-253]()
<!-- anchor: function migrateStandardProvider(info: ConfigProviderV1.Info) { … settings: info.api ? { ...options.settings, baseURL: info.api } : info.options ? -->。所以：**V1 配置没写 `npm` 的自定义 provider，迁移后 `package` 就是空的**，于是调用即报错。这不是 V2 "变严格了"，而是 V1 本来就缺一个当时可以推断的输入。

**两种修法，按你的配置写法二选一**：

- **保持 V1 语法**：给 provider 补上 `"npm": "@ai-sdk/openai-compatible"`。注意补的是 `npm` 而不是 `package`——写 `package` 会被上面第 1 条边界静默丢掉。
- **改写为原生 V2 语法**：直接写 `"package"`，取值有两种：
  - 内建的 opencode 原生包 id，例如 `@opencode/ai/providers/openai-compatible`（内建表见 [packages/core/src/provider.ts:93-93]()
<!-- anchor: ["@opencode/ai/providers/openai-compatible", () => import("@opencode/ai/provider -->）；
  - 或 `aisdk:` 前缀的 SDK 包名。前缀由 `AISDK_PREFIX` 定义，且 `aisdk()` 已带前缀时不会重复添加 [packages/core/src/provider.ts:24-26]()
<!-- anchor: export const AISDK_PREFIX = "aisdk:" … export const aisdk = (value: string) => (isAISDK(value) ? value : `${AISDK_PREFI -->。

迁移后的配置形状（示例，provider id 与模型名按需替换）：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "providers": {
    "<provider-id>": {
      "name": "<provider-id>",
      "package": "@opencode/ai/providers/openai-compatible",
      "settings": { "baseURL": "https://<your-endpoint>/v1" },
      "models": {
        "<model-id>": {
          "name": "<model-id>",
          "capabilities": {
            "tools": true,
            "input": ["text", "image"],
            "output": ["text"]
          }
        }
      }
    }
  }
}
```

> **一个必须遵守的约束**：同一个条目要整体用一种词表。V2 能读 V1 语法，但**归一化是逐条目发生的**——混写时被丢掉的是"另一个词表"的字段（第 1 条边界），而不是报错提示你。所以要么整条 V1、要么整条 V2。

## 四、终端配置：`tui.json` 的一次性搬迁

V2 把终端配置收敛为全局单个 `cli.json`，并提供了从 V1 搬迁的逻辑：读旧的 `tui.json`，加上旧状态目录里的 `kv.json`，一起迁移后写成新的 cli 配置，成功时打日志 `migrated cli config`，并记录来源与目标路径 [packages/cli/src/config/migrate.ts:100-115]()
<!-- anchor: const legacyValue = yield* readJson(path.join(input.config, "tui.json")) … }) -->。这一步是自动的，不需要手工誊抄；但如果你的旧配置里有它不认识的字段，迁移只覆盖它认识的子集。

## 五、指令文件：只认 `AGENTS.md`

V2 的指令发现只以 `AGENTS.md` 为目标：全局文件取 `<global config>/AGENTS.md` [packages/core/src/config/plugin/instruction.ts:37-37]()
<!-- anchor: const globalFile = yield* fs.resolve(join(global.config, "AGENTS.md")) -->，项目侧沿目录向上逐级查找同名的 `AGENTS.md` [packages/core/src/config/plugin/instruction.ts:70-70]()
<!-- anchor: const walked = yield* Effect.forEach(yield* fs.up({ targets: ["AGENTS.md"], star -->。

**实践含义**：如果你的项目规约写在 `CLAUDE.md`，V2 不会回退读取它——改名为 `AGENTS.md`，或建一个软链接。

## 六、验证迁移结果

```bash
opencode --version      # 应显示 V2 版本
opencode plugin list    # 插件是否按预期加载
opencode debug config   # 归一化后的最终配置（查 package / settings / capabilities 是否落位）
opencode run --model <provider>/<model> "reply with the single word: pong"
```

`plugin` 有独立子命令用于列出已装插件 [packages/cli/src/commands/commands.ts:239-244]()
<!-- anchor: Spec.make("plugin", { … params: { -->，`debug` 用于打印配置等内部状态 [packages/cli/src/commands/commands.ts:117-117]()
<!-- anchor: Spec.make("debug", { -->。最后一条 `run` 是端到端判据：能拿到模型回复，说明 provider 的运行时包已成功解析。

## 七、回滚

V2 与 V1 不能并存，回滚是"换回包 + 恢复配置"两步：

1. 卸载 `@opencode/cli`，装回 V1 的包；
2. 把 `opencode.json` 恢复为迁移前版本。

如果只想先观察再决定，稳妥顺序是：**先备份 `opencode.json` 与全局配置目录，再换包**。`opencode.json` 的 V1→V2 归一化发生在读取路径上，是一次纯内存转换——`config/normalize.ts` 内不含任何写盘调用——所以它不会就地改写你的 `opencode.json`；会落盘的只有终端配置那一次搬迁（见第四节，写的是新的 `cli.json`）。真正需要回退的是包与插件。

## 八、本基线未核实的说法

以下说法常见于迁移讨论，但**本次未在 `v2.0.11` 源码中确认**，按需自查：

- V1 插件（如第三方 `superpowers`）在 V2 下不可用、且改文件名无效——插件 API 的兼容层未在本次核对范围内；
- LSP 配置被接受但不启动语言服务器、不提供诊断；
- V1 server API 的调用方必须改用 V2 客户端——`@opencode/client` 这个包确实存在于该基线 [packages/client/package.json:3-4]()
<!-- anchor: "name": "@opencode/client", … "version": "2.0.11", -->，但"必须迁移"这个结论未从源码核实；
- `opencode debug config` 在具体版本下的完整输出字段。

这些不影响上文的迁移步骤，但不要把它们当作已核实的结论引用。
