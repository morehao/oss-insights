# Agent 基础设施（AI Infra）

本目录收录 AI Agent 底层基础设施（SDK / 库 / 组件形态）领域的开源项目解读——不提供完整产品界面，作为组件被上层 Agent 产品消费的能力层。

| 项目 | 语言 | 说明 | 协议 | 仓库 |
|------|------|------|------|------|
| [Mem0](mem0/) | Python+TS | AI Agent 智能记忆层：V3 ADD-only 记忆管线 + 多信号检索（语义+BM25+实体），四类 Provider 插件体系，双模式部署 + MCP | Apache-2.0 | [GitHub](https://github.com/mem0ai/mem0) |
| [New API](new-api/) | Go+React | 统一 AI API 网关与资产管理：四协议(OpenAI/Claude/Gemini/Responses)互转的独立 relaykit 转换层，40+ 供应商适配器 + 渠道分发，预扣/结算/退款计费与双资金池，SQLite/MySQL/PG 三库 | AGPL-3.0 | [GitHub](https://github.com/QuantumNous/new-api) |
| [Promptfoo](promptfoo/) | TypeScript | LLM 评估与红队测试：统一评估管线 + 百种攻击插件 + 风险定级，100+ Provider 抽象，SQLite 持久化 | MIT | [GitHub](https://github.com/promptfoo/promptfoo) |
| [DeepEval](deepeval/) | Python | LLM 评测框架（类似 pytest）：50+ Metric + LLM-as-Judge 三层 schema 判官协议，GEval/DAG 双判官，Synthesizer 造数据，@observe 非侵入 OTel 追踪，pytest 插件进 CI | Apache-2.0 | [GitHub](https://github.com/confident-ai/deepeval) |
| [darwin-skill](darwin-skill/) | Markdown(Agent Skill) | Agent Skill 自主优化器：9 维评分 rubric（结构59/效果35/meta6）+ paired 同-judge 多数决棘轮 + human-in-the-loop 三层守关，基于 autoresearch + 微软 SkillLens/SkillOpt | MIT | [GitHub](https://github.com/alchaincyf/darwin-skill) |
| [Agent Client Protocol](agent-client-protocol/) | Rust | 编辑器⇄编码 Agent 标准化双向 JSON-RPC 协议：Rust 类型单一事实源 → schemars 生成 JSON Schema 四件套（v1/v2 × stable/unstable），三阶段生命周期 + capabilities 双轨演进 + RFD 治理，serde 宽容反序列化矩阵 | Apache-2.0 | [GitHub](https://github.com/agentclientprotocol/agent-client-protocol) |
| [Model Context Protocol](modelcontextprotocol/) | TypeScript+Markdown | MCP 开放协议规范：TS Schema 单一事实源 → JSON Schema/文档生成管线（draft-07→2020-12 双代），工具/资源/提示三原语 + sampling/elicitation 反向请求，SEP 治理 + sep-automation 机器人 + 无状态化演进 | MIT | [GitHub](https://github.com/modelcontextprotocol/modelcontextprotocol) |
