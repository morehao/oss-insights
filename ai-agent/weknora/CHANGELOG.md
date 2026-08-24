# 变更记录

> 本解读对 WeKnora 的跨版本变更汇总；按版本倒序。描述清楚相关变动即可：上游有官方 CHANGELOG 的参照它并结合代码变动生成，否则基于代码变动（git diff / commit 区间 / 快照比对）生成。

## [0.7.2] — 2026-08-07

> 更新依据：上游 `CHANGELOG.md`（v0.7.2）+ 源码实测（本地 WeKnora v0.7.2）。

### 新增能力

- 官方文档站 `website-docs/`：VitePress 六大分区、约 50 页、360 个 API 端点文档，含自校验脚本与示例数据。
- **KB 文件夹树**：`folder_path` 独立成列（migration `000079`），目录树可浏览/重命名/批量 re-file，无需重新解析。
- **可编辑 Chunk + 修订历史 + 自定义元数据**：`chunks.source_content/content_revision/index_status` + `chunk_revisions` 表，CAS 乐观编辑 + 截图快照 + rely 重建索引。
- **Wiki 页面修订历史 + diff + 手工编辑**：`wiki_page_revisions` 表，逐行 diff（前端渲染）+ 一键回滚。
- **`resource_urls=public`**：`internal/storageurl` 包，直接返回可加载文件 URL，每资源复用 grant。
- **Feishu Drive 数据源**：blocks API 逐块钻取新格式 docx。
- **GitLab / IMA 数据源**：两个新连接器，增量游标语义各异（commit SHA / logical_key）。
- **MCP Server 1.1.x**：切到 mcp 2.x `MCPServer` API，发布 PyPI `tencent-weknora-mcp`，工具增至约 30 个。
- AWS S3 默认凭据链、本地 HTML 上传解析、批量文档打标签、QQBot Markdown 回复。
- PR CI 工作流（app/frontend/docreader/mcp-server 四组）。

### 接口 / API 变更

- `internal/router/router.go` 从 2390 行拆分为 `routes_agent.go`/`routes_knowledge.go`/`routes_chat.go`/`routes_infra.go`/`routes_auth_tenant.go` 等，路由体按域下沉（router.go 仅编排 + 顶层免认证路由），文件代理去重。
- `modelcontext` 包合并 `llmreference`/`llmresource`；handle 术语统一。
- 统一 Wiki 操作历史——移除 Wiki Browser 独立日志（migration `000077`），收敛到 KB Activity 视图。

### Schema 与数据演进

- migrations `000075_wiki_page_revisions`、`000076_knowledge_metadata_external_id_index`、`000077_remove_wiki_log`、`000078_chunk_editing_and_custom_metadata`、`000079_knowledge_folder_path`（+ SQLite 对应 `000001/000002`）。

### 移除与废弃

- 移除 Wiki Browser 冗余 operation log。

## [0.7.1] — 2026-07-24

> 更新依据：上游 `CHANGELOG.md`。

### 新增能力

- 云之家 IM 集成、Volcengine Rerank（批处理分片）、智谱 Web Search Provider、平台级 API Key（控制平面自动化）、KB Activity 审计轨、FAQ 管理增强、Langfuse OTLP/OTel 追踪（替代旧 ingestion API）、Chat 头部操作与 Markdown 导出、Prompt-Cache 可观测性、会话渠道治理。

### 移除与废弃

- **移除 Neo4j episodic 会话记忆**，聊天不再依赖 Neo4j graph，简化部署；Neo4j 只服务文档知识图谱。
- 统一 Wiki 操作历史（与上文 v0.7.2 同源，v0.7.1 先落地）。

## [0.7.0] — 2026-07-17

> 更新依据：上游 `CHANGELOG.md`。

### 新增能力

- **Scoped Tenant API Keys + principal 模型**（migrations `000064_principal_model`、`000065_tenant_api_keys`）：细粒度能力授权、可按 KB 限定、`last_used_at` 节流，配套 `DenyAPIKeyPrincipal`/`api_key_gate` 守卫与 API Integration Playground。
- Embed 网站挂件、运行时任务队列治理、RAG 管道进度事件、多存储实例绑定、会话中 MCP OAuth。
