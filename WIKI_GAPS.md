# OSS Insights Wiki 建设清单

> 基线：`main@fc2a980`（2026-10-28）  
> 核对时间：2026-10-29  
> 范围：`/Users/morehao/Documents/study/ai-agent`、`/Users/morehao/Documents/study/ai-infra` 下的本地克隆

## 判定口径

- **已建 wiki**：仓库内存在 `<分类>/<项目>-wiki/` 目录，且已被分类 `README.md` 索引收录。
- **已有解读**：存在 `<分类>/<项目>/README.md` 等深度解读页，但没有对应的 `-wiki` 目录。
- **无文档**：仓库内没有该项目的任何文档目录。
- 深度解读不算 wiki：两者读者不同——解读面向源码级分析，wiki 面向不打算读源码的人。
- 文档归属按项目定位确定；`mem0`、`zvec-grep` 的文档在 `ai-infra`，本机源码目录也已按此归位。

## 总览

| 状态 | 数量 |
|------|------|
| 已建 wiki | 10 |
| 已有解读、缺 wiki | 38 |
| 仓库内无任何文档 | 15 |
| **合计（去重）** | **63** |

> 本机共 64 个源码目录，其中 `openclaw` 在 `ai-agent` 与 `ai-infra` 各有一份克隆，去重后为 63 个项目。

路径说明：已建 wiki 的条目，`·` 后是仓库内的文档目录；待建条目的 `·` 后默认是本机源码目录，文档入口在链接里；跨分类条目额外标注了源码路径。

## 已建 wiki（10）

- [DeepSeek-Reasonix](ai-agent/DeepSeek-Reasonix-wiki/README.md) · `ai-agent/DeepSeek-Reasonix-wiki`
- [DeepSeek Harness](ai-agent/deepseek-harness-wiki/README.md) · `ai-agent/deepseek-harness-wiki`
- [RAGFlow](ai-agent/ragflow-wiki/README.md) · `ai-agent/ragflow-wiki`
- [Mem0](ai-infra/mem0-wiki/README.md) · `ai-infra/mem0-wiki`
- [Agent Client Protocol](ai-infra/agent-client-protocol-wiki/README.md) · `ai-infra/agent-client-protocol-wiki`
- [Model Context Protocol](ai-infra/modelcontextprotocol-wiki/README.md) · `ai-infra/modelcontextprotocol-wiki`
- [Hindsight](ai-infra/hindsight-wiki/README.md) · `ai-infra/hindsight-wiki`
- [EverOS](ai-infra/EverOS-wiki/README.md) · `ai-infra/EverOS-wiki`
- [Engram](ai-infra/engram-wiki/README.md) · `ai-infra/engram-wiki`
- [MOVO](ai-agent/movo-wiki/README.md) · `ai-agent/movo-wiki`

## 待建：已有解读（38）

条目按现有解读的体量降序，前面的更适合优先转成 wiki。

### ai-agent（30）

- [ ] [Multica](ai-agent/multica/README.md) · 13 页 · `ai-agent/multica`
- [ ] [RoadMan](ai-agent/roadman/README.md) · 13 页 · `ai-agent/roadman`
- [ ] [Arkloop](ai-agent/Arkloop/README.md) · 12 页 · `ai-agent/Arkloop`
- [ ] [Codeg](ai-agent/codeg/README.md) · 11 页 · `ai-agent/codeg`
- [ ] [CompozyOS](ai-agent/compozy/README.md) · 11 页 · `ai-agent/compozy`
- [ ] [Goose](ai-agent/goose/README.md) · 11 页 · `ai-agent/goose`
- [ ] [GoRaven](ai-agent/goraven/README.md) · 11 页 · `ai-agent/goraven`
- [ ] [OpenHands](ai-agent/openhands/README.md) · 10 页 · `ai-agent/openhands`
- [ ] [TripStar](ai-agent/tripstar/README.md) · 10 页 · `ai-agent/tripstar`
- [ ] [WebBrain](ai-agent/webbrain/README.md) · 10 页 · `ai-agent/webbrain`
- [ ] [WeKnora](ai-agent/weknora/README.md) · 10 页 · `ai-agent/weknora`
- [ ] [AionUi](ai-agent/aionui/README.md) · 9 页 · `ai-agent/aionui`
- [ ] [OpenWorker](ai-agent/openworker/README.md) · 9 页 · `ai-agent/openworker`
- [ ] [Tutti](ai-agent/tutti/README.md) · 9 页 · `ai-agent/tutti`
- [ ] [WorkDSH](ai-agent/workdsh/README.md) · 9 页 · `ai-agent/workdsh`
- [ ] [Eigent](ai-agent/eigent/README.md) · 8 页 · `ai-agent/eigent`
- [ ] [Coze Studio](ai-agent/coze-studio/README.md) · 7 页 · `ai-agent/coze-studio`
- [ ] [dsh-TUI](ai-agent/dsh-tui/README.md) · 7 页 · `ai-agent/dsh-tui`
- [ ] [FastGPT](ai-agent/fastgpt/README.md) · 7 页 · `ai-agent/fastgpt`
- [ ] [OpenVort](ai-agent/openvort/README.md) · 7 页 · `ai-agent/openvort`
- [ ] [OpenWork](ai-agent/openwork/README.md) · 7 页 · `ai-agent/openwork`
- [ ] [zacp](ai-agent/zacp/README.md) · 7 页 · `ai-agent/zacp`
- [ ] [AgenticSeek](ai-agent/AgenticSeek/README.md) · 6 页 · `ai-agent/AgenticSeek`
- [ ] [OpenViking](ai-agent/OpenViking/README.md) · 6 页 · `ai-agent/OpenViking`
- [ ] [OpenHuman](ai-agent/openhuman/README.md) · 6 页 · `ai-agent/openhuman`
- [ ] [Cumora](ai-agent/cumora/README.md) · 5 页 · `ai-agent/cumora`
- [ ] [OpenBee](ai-agent/openbee/README.md) · 5 页 · `ai-agent/openbee`
- [ ] [OpenWiki](ai-agent/openwiki/README.md) · 5 页 · `ai-agent/openwiki`
- [ ] [OpenWiki-km](ai-agent/openwiki-km/README.md) · 5 页 · `ai-agent/openwiki-km`
- [ ] [PicoClaw](ai-agent/picoclaw/README.md) · 2 页 · `ai-agent/picoclaw`

### 文档在其他分类（3）

源码在 `ai-agent`，文档已归入 `aigc/`、`knowledge/`。

- [ ] [DeepWiki-Open](aigc/deepwiki-open/README.md) · 4 页 · 源码 `ai-agent/deepwiki-open`
- [ ] [ComfyUI](aigc/comfyui/README.md) · 3 页 · 源码 `ai-agent/ComfyUI`
- [ ] [Memos](knowledge/memos/README.md) · 1 页 · 源码 `ai-agent/memos`

### ai-infra（5）

- [ ] [zvec-grep](ai-infra/zvec-grep/README.md) · 9 页 · `ai-infra/zvec-grep`
- [ ] [DeepEval](ai-infra/deepeval/README.md) · 7 页 · `ai-infra/deepeval`
- [ ] [New API](ai-infra/new-api/README.md) · 7 页 · `ai-infra/new-api`
- [ ] [Promptfoo](ai-infra/promptfoo/README.md) · 6 页 · `ai-infra/promptfoo`
- [ ] [Vanna](ai-infra/vanna/README.md) · 1 页 · `ai-infra/vanna`

## 待建：无任何文档（15）

这些项目需要先补深度解读，再评估是否转 wiki。

### ai-agent（9）

- [ ] **AgentsMesh** · `ai-agent/AgentsMesh`
- [ ] **OpenMAIC** · `ai-agent/OpenMAIC`
- [ ] **OpenManus** · `ai-agent/OpenManus`
- [ ] **helix** · `ai-agent/helix`
- [ ] **openclaw** · `ai-agent/openclaw、ai-infra/openclaw` · 两份克隆均无文档
- [ ] **openmanus-go** · `ai-agent/openmanus-go`
- [ ] **pi** · `ai-agent/pi`
- [ ] **react-agent** · `ai-agent/react-agent` · 克隆不完整，无 README
- [ ] **zeroclaw** · `ai-agent/zeroclaw`

### ai-infra（6）

- [ ] **go-openai** · `ai-infra/go-openai`
- [ ] **llmproxy** · `ai-infra/llmproxy` · 无 git remote，疑似本地项目
- [ ] **magpie** · `ai-infra/magpie`
- [ ] **ollama** · `ai-infra/ollama`
- [ ] **skills** · `ai-infra/skills`
- [ ] **supermemory** · `ai-infra/supermemory`

## 本机目录归属修正（2026-10-29）

| 项目 | 操作 | 原因 |
|------|------|------|
| zvec-grep | `study/ai-agent/zvec-grep` → `study/ai-infra/zvec-grep` | 文档在 `ai-infra`；本地混合检索层属于基础设施 |
| mem0 | 删除 `study/ai-agent/mem0` | `ai-infra/mem0` 已有较新克隆（10-26）；旧克隆（10-14）工作区干净、无未推送提交 |

尚未处理的归属差异：

- `ComfyUI`、`DeepWiki-Open`、`Memos` 的源码仍在 `study/ai-agent`，文档分别在 `aigc/`、`aigc/`、`knowledge/`；如需完全对齐可再移动。
- `openclaw` 在 `ai-agent` 与 `ai-infra` 各有一份克隆，均无文档，暂未合并。

## 建议顺序

1. **第一批**：已有解读且 ≥10 页的 12 个项目——MOVO、Multica、RoadMan、Arkloop、Codeg、CompozyOS、Goose、GoRaven、OpenHands、TripStar、WebBrain、WeKnora。
2. **第二批**：其余已有解读的 27 个项目。
3. **第三批**：15 个无文档项目，先做解读，再决定是否建 wiki。

## 维护方式

- 完成一个 wiki：勾选对应条目，把链接补进「已建 wiki」，并更新总览数字。
- 新建 wiki 统一放在 `<分类>/<项目>-wiki/`，并在该分类的 `README.md` 索引表加一行。
- 本清单以本机克隆与仓库索引为准；项目增删后重新核对一遍。
