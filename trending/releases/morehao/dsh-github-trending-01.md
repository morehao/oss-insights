# morehao/dsh-github-trending

> 发版记录 · 每周一次 · 最近更新 2026-11-08

## 2026-11-02 ~ 2026-11-08

- [v0.9.14](https://github.com/morehao/dsh-github-trending/releases/tag/v0.9.14) · 11-07

  v0.9.14 是一次以核心版本前移为主的小版本，自身功能无新增，主要把内置核心升级到 0.12.2 并同步文档。
  
  **主要变更**
  - 核心版本 dsh.core.version 由 0.12.0 提升到 0.12.2；首次运行会自动下载并校验 0.12.2，最低核心要求仍保持 0.12.0 不变。
  - 随 0.12.2 带入上游两个发版订阅相关修复：一是订阅仓库的变更分析此前因 legacy 游标被滚动 tag 卡住而从不落盘，现已修复；二是补发改按发布日期选取，滚动预发布不再混入。
  - 文档 docs/reference.md 同步更新到 0.9.14。
  - 自动生成的变更记录中出现的「可配置摘要输出语言」（#152）属于 v0.9.13 已发布内容，因该版本 tag 提交不在 main 历史中而被自动汇总重复列出，并非本版新增。
  
  **破坏性变更**
  - 未提及破坏性变更；最低核心版本要求未提高，仍为 0.12.0。
  
  **升级建议**
  - 若正受订阅仓库变更分析不落盘、或补发结果混入滚动预发布的问题影响，建议升级；其余用户升级风险较低，属常规跟进版本。
- [v0.9.13](https://github.com/morehao/dsh-github-trending/releases/tag/v0.9.13) · 11-06

  v0.9.13 主要新增了可配置的「摘要语言」，同时把核心与契约的配套版本要求一起抬高。
  
  **主要变更**
  - 新增「摘要语言」设置：日报与订阅摘要的输出语言不再固定为中文。默认跟随宿主 locale（zh 输出中文、en 输出英文，中英之外一律回落中文），也可在「设置 → 摘要与用量」里显式选择中文或 English。
  - 语言解析顺序为「显式设置 > 宿主 locale.preference（归一主标签）> 中文兜底」，每次运行实时读取宿主偏好，并在运行时写入核心 YAML 的 language.outputLanguage。
  - 设置页新增语言下拉项；/state 新增 language 信息，包含 hostLanguage、resolvedLanguage、outputLanguageSource。
  - 修复「榜单(scope)」tab 内 summarize / schedule 编辑不持久化的问题。
  
  **破坏性变更**
  - 核心最低版本提升至 0.12.0，低于该版本会判定为 CORE_INCOMPATIBLE 并给出下载引导；核心 0.12.0 起才支持 language.outputLanguage 与 digest.summary / summaryLang。
  - 契约包 @morehao/trendradar-contract 升级到 0.7.0，wire 层为 ipcSchema=4、digestSchema=3、contractVersion=0.7.0；摘要字段由 summaryZh 改为 summary 加 summaryLang，消费旧字段的下游需要同步适配。
  
  **升级建议**
  - 升级本版本时需同时把核心升到 0.12.0 及以上，并按契约 0.7.0 调整摘要字段的读取逻辑，否则可能遇到不兼容判定或字段缺失。
  - 有英文摘要需求、或受 scope tab 编辑不持久化影响的用户值得升级；纯中文场景且下游尚未适配新契约的，可等核心与消费方一并就绪后再升。
- [v0.9.12](https://github.com/morehao/dsh-github-trending/releases/tag/v0.9.12) · 11-06

  v0.9.12 是一个修复发行流程缺陷的补丁版本：它修复了发版注记正文的组装问题，并重新发布以补齐 Release。
  
  **主要变更**
  - 新增 `make-tag-notes.mjs`，在打 tag 前组装并校验注记正文，检查三项：首行为 `release: `、正文非空、拒绝把文件路径当作正文。
  - 该脚本与 `release-notes.mjs` 互为镜像，并纳入 `npm run check`，从而在本地阶段就能发现注记正文错误。
  - 解决了过去要等 CI 失败才发现问题、而 tag 已不可更改的痛点（#151）。
  
  **破坏性变更**
  - 说明中未提及任何破坏性变更，属于内部发版工具链与校验流程的调整。
  
  **升级建议**
  - 对使用者无 API 或功能影响，无需强制升级；若你关注发版流程的可靠性，可跟进此版本。

