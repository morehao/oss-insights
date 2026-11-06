# infiniflow/ragflow

> 发版记录 · 每周一次 · 最近更新 2025-12-07

## 2025-12-01 ~ 2025-12-07

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
- [nightly](https://github.com/infiniflow/ragflow/releases/tag/nightly) · 12-01

  本版本为 nightly 构建，未提供变更说明，无法判断具体内容。
  
  **主要变更**
  - 版本说明仅记录该 nightly 由提交 3c6d2255af06164f0818d0959af10de0afd8b8fa 构建生成。
  - 未列出任何新增功能、改进或缺陷修复条目。
  
  **破坏性变更**
  - 说明中未提及破坏性变更，但因缺少变更清单，无法确认是否存在不兼容调整。
  
  **升级建议**
  - 该版本为夜间构建，且无变更说明可供评估；不建议在生产环境直接升级。
  - 如需采用，建议先核对对应提交的代码差异并在测试环境验证。
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

