# Agent 基础设施（AI Infra）

本目录收录 AI Agent 底层基础设施（SDK / 库 / 组件形态）领域的开源项目解读——不提供完整产品界面，作为组件被上层 Agent 产品消费的能力层。

| 项目 | 语言 | 说明 | 协议 | 仓库 |
|------|------|------|------|------|
| [Mem0](mem0/) | Python+TS | AI Agent 智能记忆层：V3 ADD-only 记忆管线 + 多信号检索（语义+BM25+实体），四类 Provider 插件体系，双模式部署 + MCP | Apache-2.0 | [GitHub](https://github.com/mem0ai/mem0) |
| [New API](new-api/) | Go | 统一 AI API 网关，聚合 40+ AI 供应商，提供统一接口、计费、权限管理 | AGPL-3.0 | [GitHub](https://github.com/QuantumNous/new-api) |
| [Promptfoo](promptfoo/) | TypeScript | LLM 评估与红队测试：统一评估管线 + 百种攻击插件 + 风险定级，100+ Provider 抽象，SQLite 持久化 | MIT | [GitHub](https://github.com/promptfoo/promptfoo) |
| [DeepEval](deepeval/) | Python | LLM 评测框架（类似 pytest）：50+ Metric + LLM-as-Judge 三层 schema 判官协议，GEval/DAG 双判官，Synthesizer 造数据，@observe 非侵入 OTel 追踪，pytest 插件进 CI | Apache-2.0 | [GitHub](https://github.com/confident-ai/deepeval) |
