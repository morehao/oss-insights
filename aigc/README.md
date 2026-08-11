# AI 生成（AIGC）

本目录收录 AI 内容生成（AIGC）领域的开源项目解读——输入源数据/提示，产出图像、视频、文档等内容的引擎与平台。

| 项目 | 语言 | 说明 | 协议 | 仓库 |
|------|------|------|------|------|
| [ComfyUI](comfyui/) | Python | 模块化 AI 内容生成引擎，节点图工作流，多模态（图像/视频/音频/3D），20+ 模型架构，多策略执行缓存 + VRAM 状态机，APTO 优先设计 | GPL-3.0 | [GitHub](https://github.com/comfyanonymous/ComfyUI) |
| [Canvas2](canvas2/) | JS + React + PostgreSQL | 面向花型/图案设计的 AI 生图 SaaS：多租户 UIN 身份模型，计费积分（订单状态机 + 幂等账本 + 批次 FEFO），多渠道生图抽象（RunningHub/CFAPP→ComfyUI），18 迁移数据模型 | 私有 | 本地 /Users/morehao/Documents/works/yangu/muting/canvas2 |
| [DeepWiki-Open](deepwiki-open/) | Python+TS | 开源代码仓库 Wiki 自动生成平台，Next.js+FastAPI+AdalFlow 编排，9 种 LLM Provider，RAG+Deep Research+Codemap 引用落点 | MIT | [GitHub](https://github.com/AsyncFuncAI/deepwiki-open) |
