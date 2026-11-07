# infiniflow/ragflow

> 发版记录 · 每周一次 · 最近更新 2026-10-25

## 2026-10-19 ~ 2026-10-25

- [v1.0.0](https://github.com/infiniflow/ragflow/releases/tag/v1.0.0) · 11-06

  RAGFlow v1.0.0 是一次以功能扩展、解析与模型稳定性修复为主的版本发布。
  
  **主要变更**
  - 新增 Wally 模型提供商和 AnySearch 网页搜索提供商，扩展模型与搜索接入。
  - 新增 Vastbase G100 作为 Go 侧元数据库支持，并补充文件 API 审计钩子和日志钩子。
  - 管理端增加用户搜索、状态筛选、排序和 CLI 分页；聊天消息列表改为虚拟化以提升性能。
  - 解析侧支持将 Markdown/HTML 远程图片作为媒体引用处理，并修复 PDF 目录引导线混入正文、PDF 子分块互相覆盖、MinerU JSON 内联压缩包图片字节等问题。
  - 模型与工具侧修复 Anthropic 工具参数结构 JSON Schema、模型请求上下文取消、模型目录并发发布竞态，以及 DeepL/QWeather 凭据从节点配置或环境变量读取。
  - 其他修复包括 Dropbox 下载失败保留检查点、datasets API 错误响应体结构、PaddleOCR 提交请求错误处理、工具 HTTP 响应体限制 4 MiB、Dify 空元数据条件保持检索不限定范围等。
  
  **破坏性变更**
  - 发布说明未明确标注破坏性变更。
  
  **升级建议**
  - 本次包含较多功能新增和稳定性修复，涉及解析、模型接入、管理端与工具调用的用户建议升级；升级前仍应结合自身部署的元数据库、模型提供商和解析流程做回归验证。
- [v1.0.0-rc3](https://github.com/infiniflow/ragflow/releases/tag/v1.0.0-rc3) · 10-30

  v1.0.0-rc3 是 v1.0.0 正式版前的第三个也是最后一个候选版本，在修复 rc2 问题的同时补充了新的 Agent 能力与管道兼容性。
  
  **主要变更**
  - 为 Agent 工具循环引入用于判断证据是否充分的决策模型，并新增 Categorize 组件。
  - 摄入管道 DSL 支持多个版本。
  - 聊天历史现在会存储并展示提示词、token 使用量与执行时间。
  - Go 侧补全 Mistral /v1/ocr 客户端，并将 mistral-ocr-latest 列为 OCR 模型。
  - 解析器引入类型化的 DocItem Position API，图片摄入支持抽取 SVG 文本。
  - 其他增强包括权限 schema 字段迁移、agentic RAG 的 token 使用统计等。
  
  **主要修复**
  - 修复使用 CAJAL 科学论文助手模板创建的 Agent 多出检索节点的问题。
  - 修复 Agent webhook 头字段的保留与解析，以及增量轮询下 webhook 完成状态的保留。
  - 修复 Agent 消息中引用内容显示不正确的问题，以及 UserFillUpForm 类型相关报错。
  - 修复 S3 兼容存储未遵循 signature_version 与 addressing_style 的问题。
  - 修复 MCP tools/list 未返回私有 cacheScope、按 id 与 kb_id 删除数据集元数据、数据集搜索结果未暴露 compile_kwd 等问题。
  - 修复 chunker 在 title 家族中未保留结构化表格和图片、内存过滤未保留字符串数组、kvrocks 未传播 SetNX 错误等问题。
  - 统一 PDF 表格渲染路径，移除 RowsToHTML，改用 RenderTableHTML；并补充摄入提取器超时环境变量的文档与校验。
  
  **破坏性变更**
  - 说明中未提及破坏性变更，且明确兼容 v1.0.0-rc2 schema。
  
  **升级建议**
  - 已使用 rc2 的环境可直接升级，无需迁移。
  - 作为正式版前最后一个候选版本，建议在升级前于测试环境验证 Agent、摄入管道与聊天历史相关功能。
- [v0.27.3](https://github.com/infiniflow/ragflow/releases/tag/v0.27.3) · 10-23

  v0.27.3 以 Go 语言版 Agentic RAG 迁移为主线，同时集中修复了检索翻页、权限鉴权、存储与前端等多处缺陷。
  
  **主要变更**
  - 将 Agentic RAG 相关能力移植到 Go 实现，并补充 Go 集成测试。
  - 流水线中的解析算子文件现在支持删除。
  - 统一 ONNX Runtime native 版本至 1.29.0。
  - 前端体验改进：修正 TreeSelect 文字颜色、SelectWithSearch 下拉文字被勾选图标遮挡，以及进入 wiki 时误显示已修改等问题。
  
  **主要修复**
  - 检索与 MCP：传递 rerank_candidates_count，使结果超过 64 条时仍能正常分页；改用解析后的 embedding 模型比对检索知识库；翻译 MCP 工具 inputSchema.properties；wiki 将 MAP 抽取的分块视为缓存命中。
  - 安全与权限：任务取消改为按数据集访问权限鉴权；SSRF 防护中解开 IPv6 过渡地址。
  - 稳定性：避免嵌套编译池导致的嵌入死锁；S3 瞬时失败自动重试；容忍不同运行时的非法 header；文档更新时校验不可变字段；structure-graph 从原始行发现 bucket；修复 DOCX 预览无限更新循环、dataflow 标量文本输出形状、figure_parser 命名参数、SDK 中 agent 回答引用丢失等问题。
  - 回退了“允许重新解析在进度 0 时被取消的文档”这一改动。
  - 清理：移除废弃函数、移除未使用的知识编译代码、移除 Go artifact 删除接口。
  
  **破坏性变更**
  - 版本说明未显式标注破坏性变更，但包含若干删除性改动（废弃函数、知识编译代码、Go artifact 删除 API），依赖这些接口的集成需留意；被回退的解析行为也恢复到此前状态。
  
  **升级建议**
  - 本次多为检索分页、权限鉴权与运行稳定性方面的实际修复，建议升级；若使用被移除的接口或依赖被回退的解析行为，升级前请先做验证。

