# ComfyUI 架构深度分析

> **关联文档:** [主解读](./README.md) · [执行引擎深度分析](./execution-engine.md)

---

## 一、整体架构分层

ComfyUI 采用严格的分层架构，从底向上依次是：

```
┌──────────────────────────────────┐
│        1. Node 层                │  节点定义与注册
├──────────────────────────────────┤
│        2. Core 层                │  模型管理、执行内核
├──────────────────────────────────┤
│        3. Execution 层           │  图编排、缓存、验证
├──────────────────────────────────┤
│        4. Server / API 层        │  HTTP、WS、路由
├──────────────────────────────────┤
│        5. Frontend 层            │  独立前端仓库
└──────────────────────────────────┘
```

各层之间的依赖方向是单向的：Frontend → Server/API → Execution → Core → Nodes。Node 层虽然是底部，但它在启动时被加载到 NODE_CLASS_MAPPINGS 中供其他层引用。

---

## 二、各层模块职责

### 1. Node 层 — 节点定义与注册

**入口文件:** `nodes.py`（~2576 行）

管理所有内置节点和自定义节点：

- `NODE_CLASS_MAPPINGS` — 全局节点注册字典
- `init_extra_nodes()` — 加载 `comfy_extras/`、`comfy_api_nodes/`、`custom_nodes/` 中的节点
- 节点约定：`INPUT_TYPES()`、`RETURN_TYPES`、`FUNCTION`、`CATEGORY`

**节点来源:**

| 来源 | 数量 (文件数) | 说明 |
|------|--------------|------|
| `nodes.py` | 内联 | 核心节点（loaders、samplers、VAE、image ops 等） |
| `comfy_extras/` | 131 | 扩展节点（模型特定、video、audio、logic、math 等） |
| `comfy_api_nodes/` | ~38 | 外部 API 节点（OpenAI、Gemini、BFL、Kling 等） |
| `custom_nodes/` | 用户安装 | 社区自定义节点 |

### 2. Core 层 — 核心库

**目录:** `comfy/`（21+ 子目录，66 个顶层文件）

**模型管理:**

| 文件 | 职责 | 行数 |
|------|------|------|
| `model_management.py` | GPU/CPU 设备管理、VRAM 状态机、模型加载/卸载 | ~2067 |
| `model_patcher.py` | ModelPatcher 包装类、LoRA patching、device 管理 | ~2130 |
| `model_base.py` | 所有支持架构的基模型类 | ~2627 |
| `model_detection.py` | 从 state dict keys 自动检测模型架构 | ~1529 |
| `supported_models.py` | 支持的模型架构注册表（SD15、SDXL、FLUX 等） | ~2473 |
| `sd.py` | 模型加载管线（checkpoint、VAE、CLIP、LoRA、ControlNet） | ~2215 |

**执行相关:**

| 文件 | 职责 |
|------|------|
| `ops.py` | 自定义操作（optimized attention、quantized ops） | ~1607 |
| `quant_ops.py` | comfy-kitchen 量化操作集成 | ~287 |
| `samplers.py` | 扩散采样器实现 | ~1453 |
| `sample.py` | 采样辅助（噪声准备、latent 操作） | ~81 |
| `controlnet.py` | ControlNet 支持 |
| `lora.py` | LoRA 权重合并与管理 |

**模型实现（`comfy/ldm/`）:**

| 子目录 | 模型架构 |
|--------|---------|
| `flux/` | FLUX 模型 |
| `hunyuan_video/` | Hunyuan Video |
| `hunyuan3d/` | Hunyuan 3D |
| `wan/` | Wan 2.1/2.2 视频模型 |
| `sd3/` | SD3/SD3.5 |
| `cascade/` | Stable Cascade |
| `genmo/` | Mochi 视频模型 |
| `lightricks/` | LTX-Video |
| `modules/diffusionmodules/` | MMDiT 等共享组件 |

**文本编码器（`comfy/text_encoders/`）:** 66 个文件，覆盖 CLIP、T5、FLUX、SD3 等

### 3. Execution 层 — 执行引擎

**文件:** `execution.py` + `comfy_execution/`（9 个文件）

| 文件 | 职责 |
|------|------|
| `execution.py` | PromptExecutor 主类、PromptQueue 队列管理 |
| `comfy_execution/graph.py` | DynamicPrompt 动态图管理 |
| `comfy_execution/caching.py` | 多策略缓存系统 |
| `comfy_execution/validation.py` | 节点输入类型验证 |
| `comfy_execution/jobs.py` | 任务管理（get、cancel、validate） |
| `comfy_execution/progress.py` | 进度追踪与 WS 推送 |
| `comfy_execution/utils.py` | 执行上下文管理 |
| `comfy_execution/cache_provider.py` | 外部缓存提供者集成 |
| `comfy_execution/asset_enrichment.py` | 输出资产元数据丰富 |

### 4. Server / API 层

**入口文件:** `server.py`（~1479 行）

`class PromptServer` 基于 aiohttp：

- WebSocket 实时通信（进度、预览图、执行状态）
- PromptQueue 管理
- 中间件栈：CORS / 安全 / 弃用警告 / 压缩
- 路由组：`/ws` `/prompt` `/queue` `/history` `/upload/image` `/view` `/models` `/internal/*`

**API 层（`api_server/`）:**

- `routes/internal/internal_routes.py` — 前端专用内部端点
- `services/terminal_service.py` — 终端日志 WS 流式推送

**应用层（`app/`）:**

| 文件 | 职责 |
|------|------|
| `user_manager.py` | 用户管理、多用户支持 | ~486 |
| `model_manager.py` | 模型文件浏览 | ~221 |
| `frontend_management.py` | 前端版本管理 | ~448 |
| `custom_node_manager.py` | 自定义节点管理 |
| `assets/` | 资产管理系统（扫描、哈希、API、数据库模型） |
| `database/` | SQLAlchemy ORM + Alembic 迁移 |

### 5. Frontend 层

独立仓库 `ComfyUI_frontend`，作为 pip 包分发：

- `comfyui-frontend-package==1.47.10`
- `comfyui-workflow-templates==0.11.19`
- `comfyui-embedded-docs==0.5.9`

通过 WebSocket + HTTP 与后端通信，二进制协议用于图像预览：

```python
# protocol.py
class BinaryEventTypes:
    PREVIEW_IMAGE = 1
    UNENCODED_PREVIEW_IMAGE = 2
    TEXT = 3
    PREVIEW_IMAGE_WITH_METADATA = 4
```

---

## 三、启动流程

```
main.py 启动序列:

1. CLI 参数解析 (comfy.cli_args)
2. 环境初始化 (device 选择、deterministic、cuda_malloc)
3. 自定义路径 (model、output、input、user 目录)
4. prestartup 脚本 (custom_nodes 的 prestartup_script.py)
5. start_comfyui():
   ├─ PromptServer 创建 (aiohttp)
   ├─ nodes.init_extra_nodes() → 加载所有节点
   ├─ setup_database() → SQLAlchemy + Alembic
   ├─ prompt_server.add_routes() → 注册路由
   ├─ hijack_progress() → 进度条拦截
   └─ prompt_worker 线程启动 → asyncio 事件循环
```

---

## 四、数据流

```
用户提交 workflow JSON
        │
        ▼
PromptServer /prompt endpoint
        │
        ▼
PromptQueue 入队
        │
        ▼
PromptWorker 线程出队
        │
        ▼
DynamicPrompt 构建执行图
  ├─ 拓扑排序
  ├─ 输入验证
  └─ 缓存检查
        │
        ▼
PromptExecutor 逐节点执行:
  ├─ 检查缓存命中 → 跳过或执行
  ├─ 模型加载/卸载 (VRAM 状态机)
  ├─ 节点 forward()
  ├─ 缓存结果
  ├─ WebSocket 推送进度/预览
  └─ 输出结果收集
        │
        ▼
结果存储 (history / assets)
        │
        ▼
WebSocket 通知前端
```

---

## 五、API 设计

**路由结构:**

| 端点 | 方法 | 用途 |
|------|------|------|
| `/ws` | WebSocket | 实时执行通信 |
| `/prompt` | POST | 提交工作流 |
| `/queue` | GET/POST/DELETE | 队列管理 |
| `/history` | GET/DELETE | 执行历史 |
| `/upload/image` | POST | 图片上传 |
| `/view` | GET | 查看文件 |
| `/models` | GET | 模型浏览 |
| `/internal/*` | 多种 | 前端专用端点 |
| `/api/assets*` | 多种 | 资产管理 |

**OpenAPI 规范:** `openapi.yaml`（5160 行），定义了资产模式、队列管理、历史查询等完整 API。

**API 版本管理（`comfy_api/`）:**

- `latest/` — 最新 API（输入/输出类型系统、UI 辅助）
- `v0_0_1/`, `v0_0_2/` — 遗留版本
- `feature_flags.py` — 客户端/服务器能力协商

---

## 六、数据库设计

**ORM:** SQLAlchemy 2.0 + SQLite（默认）

**模型定义:** `app/database/models.py`

**迁移管理:** `alembic_db/` + `alembic.ini`

**关键数据模型:**

- 资产（Asset）— 用户拥有的文件，含哈希、大小、MIME 类型、预览 URL、标签
- 执行历史和结果元数据

---

## 七、关键设计模式

### 1. 自定义操作注入

模型通过 `operations` 参数接收操作实现，实现模型代码与量化/注意力优化的解耦：

```
模型代码 (forward) → operations.Linear
                        ↓
              comfy.ops / comfy-kitchen
                        ↓
              CUDA / ROCm Triton / CPU
```

### 2. ModelPatcher 包装器

所有模型被包装为 `ModelPatcher` 实例，统一管理：

- 设备移动 (`to()`, `cpu()`, `cuda()`)
- LoRA 补丁应用/移除
- 模型加载/卸载策略
- 扩展钩子（`patcher_extension.py`）

### 3. 特征标志系统

`comfy_api/feature_flags.py` 实现客户端/服务器能力协商：

- WebSocket 连接时协商
- 渐进式功能启用（如新的预览方法、输出类型）
- 前后端独立演进

### 4. 自定义节点加载器

`init_extra_nodes()` 遍历多个来源：

```
custom_nodes/   → 用户安装的节点
comfy_extras/   → 官方扩展节点
comfy_api_nodes/ → 外部 API 节点
```

每个节点可以定义 `prestartup_script.py`（启动前执行）和标准节点类。
