# 9 维评估体系深度解读

> 关联：[`README.md`](README.md) · darwin-skill 的主文档「评估 Rubric」章节（`SKILL.md:30-88`）对应的深挖。

## 一句话

darwin-skill 用一套 **9 维加权 rubric**（结构 59 + 效果 35 + meta 6，总分 100）把"一个 Skill 好不好"从主观印象变成可量化的打分；但更关键的设计是**承认 LLM judge 打分不可靠**，因此把绝对分数降级为 triage（决定先改谁），把真正判断真进步与否交给 paired 比较（见 [`paired-ratchet.md`](paired-ratchet.md)）。

## 维度构成：结构 / 效果 / meta 三层

rubric 的 9 个维度权重并非拍脑袋，而是三块性质不同的评估拼起来：

| 层级 | 维度 | 权重 | 解决的问题 |
|------|------|------|-----------|
| 结构（静态分析） | 1 Frontmatter 质量 | 7 | 入口信息（name/description/触发词）是否完整规范 |
| 结构 | 2 工作流清晰度 | 12 | 步骤是否可执行、有序号、有明确输入输出 |
| 结构 | 3 失败模式编码 | 12 | 是否写"如果 X 失败 → Y"分支，不只写正向流程 |
| 结构 | 4 检查点设计 | 6 | 关键决策前是否有显性用户确认（🔴/STOP） |
| 结构 | 5 可执行具体性 | 18 | 是否禁绝"建议/可以考虑"等模糊措辞 |
| 结构 | 6 资源整合度 | 4 | references/scripts 路径是否可达 |
| 效果（需实测） | 7 整体架构 | 12 | 结构层次、冗余、是否含 AI 腔废话段 |
| 效果 | 8 实测表现 | 23 | 用测试 prompt 跑一遍，输出是否达到宣称能力 |
| meta | 9 反例与黑名单 | 6 | 是否有"不要做什么"的反例清单 |

三个观察：

- **效果维权重最高（23+12=35），结构维权重分散**：说明作者的核心信念是"Skill 写得再漂亮，跑出来效果不好就是零"（`README.md:148`）。实测表现一个维度就占 23 分，是整个 rubric 最大单点。
- **新增的三个维度直指 SkillLens 实证**：dim3（失败模式编码）、dim5（可执行具体性）、dim9（反例黑名单）正是 SkillLens 论文里把 LLM-judge 准确率从 46.4% 拉到 73.8% 的三个 meta-skill 维度 [SKILL.md:32-32]()。

<!-- snippet: > **设计依据**：基于 SkillLens 论文（arXiv 2605.23899）实证发现——LLM-as-judge 评估 skill 质量准确率仅 46.4%（接近随机） -->
- **部分维度是"负向惩罚"而非"正向加分"**：dim6 资源整合度看"路径可达"、dim7 看"冗余/AI 腔出现扣分"，dim5 的「禁止模糊词」更是典型的 must-not-have 语义——它给的是约束，不是能力。

## 「总得分」的真实用途：只有 triage，不能当棘轮

rubric 最反直觉的一条写在评分规则里——**绝对总分只用于粗排，绝不用于 keep/revert**：

> 绝对总分只用于 triage（粗排「哪支最弱、先改谁」），绝不用于 keep/revert。实测：同一份**未改**文字换个 judge 评，总分可摆 ±8（`SKILL.md:62`）。

<!-- snippet: ⚠️ **绝对总分只用于 triage -->

理由在 [`paired-ratchet.md`](paired-ratchet.md) 展开，这里核心一句：**绝对分数住在「文字 × 该 judge 当下选的标准」里，不是文字属性**。同一个文字，换个人评能差 ±8 分，这说明绝对分大部分是"评判者的尺子"而非"文字质量"。所以：

- **基线评估（Phase 1）** 用绝对分排 rank——决定先改谁（triage 语义成立，因为 judge 对 gross 差异一致）。
- **keep/revert（Phase 2）** 不能用绝对分——改用同行 judge 的 paired 比较（同 judge 一次 call 内比改前/改后，误差相减抵销）。

## LLM-judge 反偏差设计：四条配套机制

rubric 有效的前提是"打分的人可信"。SkillLens 实证 LLM 自评准确率仅 46.4%（比扔硬币还差），因此 darwin-skill 围绕"打分"铺了四条防线：

| 机制 | 做法 | 针对的偏差 | 出处 |
|------|------|-----------|------|
| 独立 judge | 评分用子 agent，主 agent 不参与打分 | 「自己改自己评」乐观偏差 | [SKILL.md:23-23]() |
| 评委不复用 | 下一轮换全新 judge | 锚定效应（同一 judge 延续上一轮标准） | [SKILL.md:163-163]() |
| 盲评 | 只给改前改后两版文字，不给"哪个是新的" | 位置/顺序偏好 | [references/skilllens-evidence.md:44-44]() |
| dry_run 控制 | 干跑比例 > 30% 告警，强制至少 1 个 full_test | 没真跑就给效果分（权重 23% 等于编造） | [SKILL.md:87-87]() |

<!-- snippet: **dry_run 比例 > 30% → 评估失效警告** -->

其中 dry_run 控制尤其值得注意：`dim8` 实测权重 23%，是全 rubric 最大单点。但实测要跑子 agent，环境有限时会退化为"干跑"（读完 skill 模拟推演）。若 30% 以上的评分都靠干跑，效果维度基本是编出来的。darwin-skill 为此设了硬阈值告警，并要求至少保留 1 次真实 full_test。

## 为什么「找最短板」要用加权短板而非原始最低分

Phase 2 每轮要挑"改哪个维度"，直觉是找原始分最低的维度。darwin-skill 明确反对：

> 找出加权短板最大的维度：weighted_gap = weight × (10 − score) / 10，结构或效果都算（`SKILL.md:193`）。

<!-- snippet: 找出加权短板最大的维度：weighted_gap = weight × (10 - score) / 10 -->

为什么？低权重维度会制造**进步幻觉**——issue #18 实战记过：dim9（权重 6，gap 5.3）原始分最低被优先修，而 dim8（权重 23）加权短板最大（11.5）却 4 轮未动 [SKILL.md:196-196]()。

<!-- snippet: # dim9（权重6，gap 5.3）原始分最低被优先修，而 dim8（权重23）加权短板最大（11.5）却 4 轮未动 -->

加权短板 = "该维度还能贡献多少总分"。修一个权重 23 的短板，对总分提升远大于修一个权重 6 的短板。多个短板加权值相近（差距 ≤1.0）时，才回退为原始分升序。

## 维度相关簇：修一个跟着涨一个

Phase 2 诊断还有一条实战教训——dim2/dim3/dim4 是**相关簇**：

> HL-3 警告：dim2/dim3/dim4 是相关簇，修一个时另两个常跟着涨（`SKILL.md:198-199`）。

<!-- snippet: HL-3 警告：dim2/dim3/dim4 是相关簇，修一个时另两个常跟着涨 -->

`references/skilllens-evidence.md` 的 huashu-gpt-image 案例证实：Round 1 攻 dim3（最低 6.5）改成 10，同期 dim2 自动从 7.5 → 9（未单独优化）；Round 2 想单独攻 dim2 却已触顶 [references/skilllens-evidence.md:106-106]()。

<!-- snippet: - Round 1 攻 dim3（最低 6.5）→ 改成 10 -->

这解释了为什么"每轮只改一个维度"（反例黑名单第 5 条）：改一个，相关簇跟着变，才能把分数变化归因到具体改动上。

## 实证基线：rubric 到底能不能识别质量差异

`references/skilllens-evidence.md` 记录了 darwin-skill 自己做的 controlled study——对 huashu-research 应用 4 类劣化（模糊化指令/删检查点/删异常表/插 AI 废话），让 5 个独立 judge 盲测：

| 维度 | V1 均值 | V2(degraded) 均值 | Δ | 一致性 |
|------|--------|-------------------|----|--------|
| 3 边界条件 | 9.2 | 3.4 | -5.8 | 全部识别 |
| 4 检查点 | 9.0 | 2.6 | -6.4 | 全部识别 |
| 8 实测 | 9.0 | 3.6 | -5.4 | 全部识别 |
| **总分** | **89.6** | **43.2** | **+46.5** | **5/5 high confidence** |

数据来源 [references/skilllens-evidence.md:51-51]()。

<!-- snippet: | 1 | V1 → V2 | 89.5 | 41.7 | **+47.8** | V1>V2 | high | -->

**结论的双面性（darwin-skill 自己明确声明）**：rubric 能识别 **gross degradation**（5/5 high confidence），但 SkillLens 的 46.4% 是来自**细粒度**对比，所以**不能证明 fine-grained 差异也能识别** [references/skilllens-evidence.md:73-73]()。

<!-- snippet: **rubric 能识别 gross degradation（5/5 high confidence）**，但**这不能证明 fine-grained quality difference 也能识别** -->

这就是为什么"重要决策必须人审"以及 keep/revert 必须走 paired 比较——rubric 在"明显差很多"时可靠，在"差不多但略有差异"时不可靠，而后者恰恰是优化循环每轮最常遇到的场景。

## 与同类评测系统的对照

| 维度 | darwin rubric | DeepEval LLM-as-Judge | promptfoo |
|------|--------------|----------------------|-----------|
| 评分对象 | Agent Skill 文档 | LLM 输出 | LLM 输出 |
| 打分方式 | 9 维 rubric + paired 比较 | Metric + Judge 协议 | 断言 + 评测 prompt |
| 权重 | 结构/效果/meta 三层加权 | metric 组合 | 可配置 |
| 反偏差 | 独立 judge + 评委不复用 + paired | schema 化抽取 | 确定性断言为主 |
| 人审 | 强制三层守关 | 无（配置走） | 无 |

darwin-skill 的独特点在于它把"评测"和"被优化系统"统一进同一个自指闭环——rubric 既评价别人，也评价它自己；而 DeepEval / promptfoo 是外部的评测工具，不反向作用于自身的优化。
