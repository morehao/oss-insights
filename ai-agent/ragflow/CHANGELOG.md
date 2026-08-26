# 变更记录

> 本解读对 RAGFlow（infiniflow/ragflow）的跨版本变更汇总；按版本倒序。描述清楚相关变动即可：若上游有官方 changelog，参照它并结合代码变动生成；没有则基于代码变动（git diff / release notes / 代码快照比对）生成。

## [dev-20260826]（2026-08-26 增量）

> 依据：本地源码仓库 `git diff`（`2db8eb6..HEAD` 区间 + 近期 commit）。本版为对初版解读（基线 `2db8eb6`）的增量更新，重点同步 Agentic RAG 重构与新增数据源连接器，并对全部子文档引用行号做了一次深度重核。

- **Agentic RAG 机制重构**：初版记录的 `internal/agentic_rag/`（ReAct 六工具：think/todo_write/grep_chunks/search_chunks/list_chunks/run_javascript）已被上游 `23a0711`（Revert "port smart-reasoning agentic RAG to eino ADK"）删除；Agentic RAG 改为 `internal/agent/harness/` 下的图式状态机（route → pre_search → planner → orchestrator → formalize_answer，mode 驱动）。
  - 影响文档：`rag-pipeline.md`、`agent-platform.md`
  - 关键新落点：`internal/agent/harness/{agentic_rag,agentic,pipeline,production,types}.go`、`internal/agent/tool/agentic_search.go`
- **新增数据源连接器**：Azure DevOps connector（仓库源码 + 拉取请求，支持 hosted/self-hosted，checkpoint/resume + commit 变更探测）、Azure Blob Storage connector（对象容器，checkpoint 续传）；均注册于 `internal/syncer/connector/builtin.go`。
  - 影响文档：`README.md`、`ingestion-deepdoc.md`
- **引用行号深度重核**：对全部 7 篇 md 重跑 `verify-references.mjs`，覆盖漂移行号（MISMATCH）、修正失效路径；`internal/agentic_rag/*` 五处失效引用改写为 `internal/agent/harness/*` 新路径。
- 其余大量近期提交为 fix 级（PDF 解析、web 样式、agent 容错、memory prompt 对齐等），不构成用户可感知的能力变化，未逐一展开。
