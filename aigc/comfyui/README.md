# ComfyUI — 模块化 AI 生成引擎与可视化工作流平台

> **GitHub:** [comfyanonymous/ComfyUI](https://github.com/comfyanonymous/ComfyUI)
> **文档:** [docs.comfy.org](https://docs.comfy.org/)
> **许可:** GPLv3 | **语言:** Python | **版本:** 0.29.0
>
> **深读文档:** [架构深度分析](./architecture.md) · [执行引擎深度分析](./execution-engine.md)

---

## 一、项目概览

ComfyUI 是一个模块化 AI 内容生成引擎，提供节点图工作流界面，让用户无需编码即可组装 AI 管线。它最初聚焦图像生成（Stable Diffusion），现已扩展支持视频、音频、3D 等多模态生成。

**核心能力:**

| 领域 | 支持的模型架构 |
|------|---------------|
| 图像 | SD1.x/2.x, SDXL, SD3/3.5, FLUX, Pixart, HunyuanDiT, AuraFlow 等 20+ |
| 视频 | SVD, Mochi, Hunyuan Video, Wan 2.1/2.2, LTX-Video, Cosmos 等 |
| 音频 | Stable Audio |
| 3D | Hunyuan3D, Gaussian Splatting, TripoSG |
| LLM 集成 | OpenAI, Gemini, Anthropic, Grok, OpenRouter 等 |

**技术栈一览:**

| 层次 | 技术选型 |
|------|---------|
| 语言 | Python >=3.10 |
| Web 框架 | aiohttp（异步） |
| 深度学习 | PyTorch（CUDA / ROCm / MPS / XPU / NPU） |
| 数据库 | SQLAlchemy 2.0 + SQLite, Alembic 迁移 |
| 量化内核 | comfy-kitchen (FP8/INT8/NVFP4) |
| 内存管理 | comfy-aimdo（动态 VRAM 控制） |
| 前端 | 独立仓库 ComfyUI_frontend，pip 包分发 |
| 测试 | pytest + pytest-aiohttp + pytest-asyncio |

---

## 二、同类对比

| 特性 | ComfyUI | Automatic1111 | InvokeAI | Forge |
|------|---------|---------------|----------|-------|
| 界面范式 | 节点图 | 标签页表单 | Web UI | Web UI |
| 模型架构支持 | 40+ | 15+ | 10+ | 20+ |
| 多模态 | 图像/视频/音频/3D | 图像 | 图像 | 图像 |
| 执行图缓存 | 智能缓存（按哈希） | 无 | 部分 | 基础 |
| 自定义节点生态 | 极大规模 | 大规模 | 中等 | 中等 |
| API 优先设计 | 是，OpenAPI | 基础 REST | 基础 | 基础 |
| 内存优化 | 动态卸载 + VRAM 状态机 | 半精度基础 | 半精度 | 半精度 |
| 离线可用 | 是 | 是 | 是 | 是 |
| 前端独立 | 是（独立仓库） | 否 | 否 | 否 |

---

## 三、项目规模

| 维度 | 数据 |
|------|------|
| GitHub Stars | 60k+ |
| Python 文件数 | ~450+ |
| 核心库 `comfy/` 子目录 | 21+ |
| `comfy_extras/` 节点文件 | 131 |
| `comfy_api_nodes/` 节点文件 | ~38 |
| openapi.yaml 行数 | 5160 |
| 测试文件数 | ~30+ |

---

## 四、架构总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Frontend (独立仓库)                           │
│              ComfyUI_frontend (pip: comfyui-frontend-package)        │
└───────────────────────────┬─────────────────────────────────────────┘
                            │ WebSocket + HTTP
┌───────────────────────────▼─────────────────────────────────────────┐
│                        Server Layer (server.py)                      │
│  PromptServer: aiohttp WebServer + WebSocket + PromptQueue           │
│  Middleware: CORS / Security / Compression / Deprecation             │
└──────┬──────────────────────────────┬────────────────────────────────┘
       │                              │
┌──────▼──────────────┐    ┌─────────▼────────────────────────────────┐
│  API Layer          │    │  Execution Layer                          │
│  (api_server/)      │    │  (execution.py + comfy_execution/)        │
│  • Internal routes  │    │  • PromptExecutor                         │
│  • Asset API        │    │  • DynamicPrompt (动态图)                 │
│  • User settings    │    │  • 多策略缓存 (LRU / RAM / 哈希)          │
│  • Terminal WS      │    │  • 节点验证 + 类型检查                    │
└─────────────────────┘    └─────────┬────────────────────────────────┘
                                      │
┌─────────────────────────────────────▼────────────────────────────────┐
│                        Core Library (comfy/)                         │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐    │
│  │ 模型检测      │  │ 模型加载      │  │ 模型管理 + 内存管理       │    │
│  │ model_detect │  │ sd.py        │  │ model_management.py      │    │
│  │ ion.py       │  │ model_base   │  │ model_patcher.py          │    │
│  │ supported_m  │  │ .py          │  │                           │    │
│  │ odels.py     │  │              │  │  VRAM 状态机              │    │
│  └──────┬───────┘  └──────┬───────┘  │  DISABLED│NO│LOW│NORMAL│  │    │
│         │                 │          │  HIGH│SHARED               │    │
│         └─────────┬───────┘          └──────────────────────────┘    │
│                   │                                                  │
│  ┌────────────────▼────────────────────────────────────────────┐    │
│  │ 模型实现 (comfy/ldm/)                                        │    │
│  │ flux/  hunyuan_video/  wan/  sd3/  cascade/  genmo/  ...    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐    │
│  │ 自定义操作     │  │ 量化内核      │  │ 采样器                    │    │
│  │ ops.py        │  │ quant_ops.py │  │ samplers.py             │    │
│  │ (attention /  │  │ (FP8/INT8    │  │ k_diffusion/            │    │
│  │  patcher ops) │  │  /NVFP4)     │  │ extra_samplers/          │    │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘    │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐    │
│  │ 文本编码器    │  │ 音频/图像     │  │ 实用工具                  │    │
│  │ text_encoder │  │ 编码器        │  │ utils / folder_paths    │    │
│  │ s/           │  │ audio_enc /  │  │ / node_helpers           │    │
│  │              │  │ image_enc    │  │                          │    │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                                      │
┌─────────────────────────────────────▼────────────────────────────────┐
│  Nodes (nodes.py + comfy_extras/ + comfy_api_nodes/ + custom_nodes/) │
│  NODE_CLASS_MAPPINGS 注册 → loaders / samplers / image / video /    │
│  audio / 3D / controlnet / LoRA / logic / math / string / API ...   │
└─────────────────────────────────────────────────────────────────────┘
```

**启动流程（main.py）:**

```
CLI 参数解析 → 环境初始化 → 自定义路径 → prestartup 脚本
→ PromptServer 创建 → extra_nodes 加载 → 数据库初始化
→ 路由注册 → 进度条劫持 → prompt_worker 线程启动 → asyncio 事件循环
```

---

## 五、核心模块导读

| 模块 | 说明 | 深读文档 |
|------|------|---------|
| 执行引擎 | 节点图编排、缓存策略、动态图扩展、异步节点 | [execution-engine.md](./execution-engine.md) |
| 架构总览 | 分层架构、模块职责、依赖链路 | [architecture.md](./architecture.md) |

---

## 六、设计亮点

### 1. 智能缓存系统

执行引擎实现了多策略缓存架构，是性能核心：

```python
# comfy_execution/caching.py — 缓存策略设计
BasicCache                 # 基础字典缓存
├── CacheKeySetID          # 按节点 ID 缓存（简单，无哈希）
├── CacheKeySetInputSignature # 按输入内容哈希缓存（精确）
│   └── HierarchicalCache  # 层级缓存（父级无效→子级无效）
└── NullCache              # 禁用缓存

LRUCache                   # LRU 淘汰策略
└── RAMPressureCache       # 内存压力感知淘汰
```

- **粒度:** 节点级缓存，仅重新计算变化部分
- **失效传播:** `HierarchicalCache` 实现父节点变化→子节点自动失效
- **内存感知:** `RAMPressureCache` 根据系统内存压力动态淘汰

### 2. VRAM 状态机

`comfy/model_management.py` 定义了精细的 VRAM 状态管理：

```
DISABLED → NO_VRAM → LOW_VRAM → NORMAL_VRAM → HIGH_VRAM → SHARED
```

每个状态对应不同的模型驻留策略，结合 `comfy-aimdo` 实现动态卸载。模型加载时根据可用 VRAM 自动选择状态，采样过程中可动态升降级。

### 3. 动态图执行

`DynamicPrompt`（`comfy_execution/graph.py`）允许在运行时动态扩展执行图：

- 节点可以在执行过程中生成新的子图（ephemeral nodes）
- 适用于：条件分支、循环、动态采样调度
- 原始图结构和动态扩展的执行图有明确的生命周期管理

### 4. 模型自动检测

`comfy/model_detection.py`（1529 行）通过读取 state dict 的 key 前缀、权重形状、元数据信息，自动识别 40+ 种模型架构及其配置（UNet 深度、cross-attention 维度、latent 格式等）。无需用户手动指定模型类型。

### 5. 量化内核抽象

`comfy/ops.py` + `comfy/quant_ops.py` 提供了统一的量化操作接口，底层由 `comfy-kitchen` 提供硬件优化实现：

- 支持 FP8 (E4M3/E5M2)、INT8、NVFP4
- 自动选择 CUDA / ROCm Triton 后端
- 模型代码不关心具体量化实现，通过 `operations` 参数注入

### 6. 节点类型系统

`comfy/comfy_types/node_typing.py` 定义了节点 IO 的类型系统：

```python
class IO(Enum):
    STRING = "STRING"
    IMAGE = "IMAGE"
    LATENT = "LATENT"
    MODEL = "MODEL"
    CLIP = "CLIP"
    VAE = "VAE"
    CONDITIONING = "CONDITIONING"
```

支持组合类型、通配符（`AnyType("*")`）、联合类型（`"FLOAT,INT"`），运行时进行严格的类型检查和自动转换。

---

## 七、总结与评价

### 亮点

- **架构清晰:** 严格的分层设计（Server → Execution → Core → Nodes），各层职责分明
- **生态丰富:** 40+ 模型架构、131 个额外节点文件、38 个 API 节点，覆盖多模态生成全场景
- **性能优化:** 多策略缓存 + VRAM 状态机 + 量化内核，精心打磨的执行效率
- **API 优先:** 完整的 OpenAPI 规范、WebSocket 实时通信、模块化的 API 版本管理
- **社区活跃:** 周更节奏、独立前端仓库、comfy-kitchen/comfy-aimdo 等配套生态

### 不足

- **项目管理:** 核心节点和插件节点的职责边界不够清晰（nodes.py ~2576 行）
- **配置分布式:** CLI 参数、YAML 配置、数据库配置、前端配置多个来源，管理路径不统一
- **Node 生态碎片化:** `comfy_extras/` 与 `comfy_api_nodes/` 存放同类节点，但物理隔离
- **文档滞后:** 内部接口文档较少，大量依赖源码阅读；OpenAPI spec 与实际路由不完全一致

### 可借鉴的设计

1. **缓存失效传播机制:** HierarchicalCache 的设计可普遍应用于有依赖关系的计算管线
2. **VRAM 状态机:** 将资源管理建模为有限状态机，各状态有清晰的升降级策略
3. **模型检测模式:** 通过 state dict 反射自动识别模型架构，而非依赖用户手动配置
4. **Ops 抽象层:** 将底层量化/注意力实现抽象为可注入的 `operations` 参数，解耦模型实现和性能优化
5. **动态图执行:** 执行时动态扩展节点图的能力，适用于条件逻辑和循环场景
