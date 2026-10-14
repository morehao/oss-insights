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
| [Agent Client Protocol 项目 Wiki](agent-client-protocol-wiki/README.md) | Markdown（文档 + 13 张 Mermaid 图） | 面向**不打算读源码的人**的 ACP 中文项目 Wiki（9 页 / 几乎零代码，基线 `v1.11.0` @ `4cf3dd8`）：协议定位与生态位（本仓库是「立法机构」而非运行时），两个角色与 JSON-RPC 消息模型（方法清单、缺省即不支持的能力协商、`_meta` / `_` 前缀扩展），一次协作的完整时间轴（initialize → session/new\|load\|resume → prompt 回合 → 取消 → close/delete，含停止原因与工具调用状态机），工具调用与权限（授权四选项、工具调用字段与 diff/终端内容、fs/terminal/elicitation 三个能力面、v2 移除 Client 执行面的取舍），Schema 生成管线（Rust 类型单一事实源 → 4 套 schema.json + meta.json + schema.mdx，注解反向测试与 CI 生成物比对），三层版本解耦与 v1→v2 变更、RFD 生命周期与 release-plz 发布链路，生态（客户端/Agent/官方与社区 SDK、注册表自动生成、实验性 TCK），以及仓库结构、构建命令与质量门禁 | Apache-2.0（同上游，文档） | [GitHub](https://github.com/agentclientprotocol/agent-client-protocol) |
| [Model Context Protocol](modelcontextprotocol/) | TypeScript+Markdown | MCP 开放协议规范：TS Schema 单一事实源 → JSON Schema/文档生成管线（draft-07→2020-12 双代），工具/资源/提示三原语 + sampling/elicitation 反向请求，SEP 治理 + sep-automation 机器人 + 无状态化演进 | MIT | [GitHub](https://github.com/modelcontextprotocol/modelcontextprotocol) |
| [zvec-grep](zvec-grep/) | TypeScript | 面向人与 Agent 的本地优先混合检索层（zg）：内嵌 ripgrep 精确/正则、zvec FTS 词法排序与向量语义召回经 RRF 融合为同一接口，tree-sitter 结构感知切分（symbol/签名/breadcrumb/行号），Model2Vec/ONNX/GGUF/远端四类 Embedding 后端 + 工作区索引，Direct/Server 双执行形态（同一 engine）+ watcher 目录级压缩与小时级对账，本地 MCP Server 默认只暴露一个搜索工具 + 六种 Agent 安装器，远端嵌入需显式数据授权 | Apache-2.0 | [GitHub](https://github.com/zvec-ai/zvec-grep) |
