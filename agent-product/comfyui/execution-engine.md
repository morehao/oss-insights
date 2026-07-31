# ComfyUI 执行引擎深度分析

> **关联文档:** [主解读](./README.md) · [架构深度分析](./architecture.md)

---

## 一、设计动机

ComfyUI 的执行引擎需要解决几个核心问题：

1. **节点图如何被高效执行？** — 用户拖拽连接节点形成 DAG，引擎需要拓扑排序并逐节点调度
2. **如何避免重复计算？** — 修改工作流中的少数节点时，应只重新计算受影响的部分
3. **如何支持动态行为？** — 节点在执行过程中可能需要生成新的子图（条件分支、循环）
4. **如何管理 GPU 内存？** — 多个大模型在有限 VRAM 中切换执行

执行引擎围绕 `execution.py` 的 `PromptExecutor` 和 `comfy_execution/` 的辅助模块给出了答案。

---

## 二、核心概念

### 执行图结构

执行图是一个有向无环图（DAG），节点代表处理单元，边代表数据流。图的拓扑次序决定了执行顺序。

```
[CheckpointLoader] ────→ [CLIPTextEncode] ────→ [KSampler] ────→ [VAEDecode] ────→ [SaveImage]
                        ↑
               [Positive Prompt]  [Negative Prompt]
```

### 节点接口约定

每个节点遵循统一的接口模式：

```python
class SomeNode:
    @classmethod
    def INPUT_TYPES(cls):
        return {
            "required": {"input_name": (IO_TYPE, {...})},
            "optional": {"opt_input": (IO_TYPE, default)},
            "hidden": {"internal_param": "VALUE"},
        }

    RETURN_TYPES = (IO_TYPE,)
    FUNCTION = "execute"
    CATEGORY = "category/path"

    def execute(self, input_name, opt_input=None, **kwargs):
        # 处理逻辑
        return (result,)
```

---

## 三、缓存系统

缓存是引擎性能核心。完整的缓存策略体系定义在 `comfy_execution/caching.py`（~602 行）。

### 类型层级

```
Cache (ABC)
├── BasicCache                     # 内存字典存储
│   └── CacheKeySetID              # key = 节点唯一 ID → 跳过哈希计算
│   └── CacheKeySetInputSignature  # key = 输入内容哈希 → 精确匹配
│       └── HierarchicalCache      # 层级传播：父变→子亦无效
│   └── NullCache                  # 禁用缓存 (调试用)
├── LRUCache                       # 固定容量的 LRU 淘汰
│   └── RAMPressureCache           # 基于系统内存压力的动态淘汰
```

### 关键设计

**1. CacheKeySetInputSignature — 精确缓存匹配**

通过计算输入的唯一签名（哈希）来决定缓存是否有效：

- 输入 hash = `hash(节点类型, 所有输入值)`
- 相同签名 → 跳过执行，复用结果
- 不同签名 → 重新执行，更新缓存

**2. HierarchicalCache — 层级失效传播**

```
节点 A 缓存失效
    │
    ▼
子节点 B 的缓存自动失效（因为 B 依赖 A 的输出）
    │
    ▼
孙节点 C 的缓存自动失效
```

这是缓存正确性的关键：确保整个依赖链在变更时被完整重计算。

**3. RAMPressureCache — 内存感知淘汰**

基于 `psutil` 监控系统内存压力：

```
内存充足 → 缓存全部保留
内存紧张 → 淘汰最久未使用的缓存条目
内存极紧 → 淘汰大部分缓存
```

### 缓存工作流

```
节点执行前:
1. 计算缓存 key（ID 或输入签名）
2. 查询缓存:
   ├─ 命中 → 直接返回缓存结果，跳过执行
   └─ 未命中 → 执行节点，缓存结果

缓存写入时:
1. 存储输出结果
2. 在 HierarchicalCache 中标记子节点需重新验证
```

---

## 四、动态图执行

**文件:** `comfy_execution/graph.py`（~342 行）

### DynamicPrompt

`DynamicPrompt` 是静态 prompt 的运行时包装器，支持图的动态扩展：

```python
class DynamicPrompt:
    def __init__(self, prompt):
        self.prompt = prompt        # 原始静态图
        self.ephemeral = {}         # 运行时生成的动态节点
        self.get_node(...)          # 统一访问静态+动态节点
```

**动态图执行流程:**

```
原始 Prompt JSON
        │
        ▼
DynamicPrompt 初始化
        │
        ▼
拓扑排序确定执行顺序
        │
        ▼
遍历节点:
  ├─ 普通节点 → 正常执行
  └─ 动态生成节点:
      ├─ 节点执行时调用 set_ephemeral_node()
      ├─ 新节点注册到 DynamicPrompt.ephemeral
      └─ 引擎继续执行新节点
```

### 动态节点生命周期

```
1. 当前节点执行中:
   node_created = set_ephemeral_node(node_id, node_def, inputs)
   
2. 引擎回调:
   dynamic_prompt.add_ephemeral(node_id, node_created)
   
3. 引擎继续遍历:
   ephemeral 节点被纳入拓扑排序和缓存系统
   
4. 清理:
   执行完成后，ephemeral 节点随 DynamicPrompt 销毁
```

---

## 五、执行流程

```
用户提交 (QueuePrompt)
    │
    ▼
PromptQueue.put() — 入队
    │
    ▼
PromptWorker 线程处理:
    │
    ├─ 1. queue.get() — 出队
    │
    ├─ 2. PromptExecutor.execute():
    │       │
    │       ├─ a. 构建 DynamicPrompt
    │       │
    │       ├─ b. 拓扑排序
    │       │
    │       ├─ c. 遍历节点:
    │       │   ├─ 缓存检查 → 命中则跳过
    │       │   ├─ 输入验证 (validation.py)
    │       │   ├─ 模型加载 (按需)
    │       │   ├─ 节点 forward()
    │       │   ├─ 模型卸载 (按需)
    │       │   ├─ 缓存结果
    │       │   └─ WS 推送进度/预览
    │       │
    │       ├─ d. 动态图扩展 (set_ephemeral_node)
    │       │
    │       └─ e. 结果收集
    │
    ├─ 3. 结果存储 (history/assets)
    │
    └─ 4. WS 通知前端执行完成
```

---

## 六、节点验证系统

**文件:** `comfy_execution/validation.py`（~58 行）

在节点执行前检查输入类型兼容性：

```python
def validate_node_input(input_value, input_type):
    # 类型匹配：
    #   IMAGE → IMAGE ✓
    #   "*" (AnyType) → 任何类型 ✓
    #   "FLOAT,INT" (Union) → FLOAT 或 INT ✓
    #   不匹配 → 抛类型错误
```

**类型系统:**

- `IO.STRING`, `IO.IMAGE`, `IO.LATENT`, `IO.MODEL` 等枚举类型
- `AnyType("*")` — 通配符，接受任意类型
- `"FLOAT,INT"` — 联合类型，接受任一
- `Combo(["opt1", "opt2", ...])` — 下拉列表，自动校验

---

## 七、异步节点支持

执行引擎支持异步节点（通过 `IS_CHANGED` 和 `VALIDATE_INPUTS` 等特殊方法）：

```
节点定义可包含:
├─ FUNCTION = "process"         → 主执行方法
├─ IS_CHANGED = "check_change" → 自定义变更检测
├─ VALIDATE_INPUTS = "validate" → 自定义输入验证
├─ OUTPUT_NODE = True           → 标记为输出节点
├─ INPUT_IS_LIST = True         → 接受列表输入
└─ CATEGORY = "path"            → 菜单分类
```

异步节点通过 `ExecutionBlocker`（`comfy_execution/graph_utils.py`）支持条件阻塞执行。

---

## 八、任务管理

**文件:** `comfy_execution/jobs.py`

管理 PromptQueue 中任务的完整生命周期:

| 操作 | 说明 |
|------|------|
| `get()` | 获取下一个待执行任务 |
| `cancel()` | 取消指定任务 |
| `validate()` | 验证任务是否可执行 |
| `get_current()` | 获取当前正在执行的任务 |

任务与 WebSocket 会话绑定，支持多用户队列。

---

## 九、进度与预览

**文件:** `comfy_execution/progress.py`

执行进度通过 WebSocket 实时推送：

```
节点执行中:
├─ 进度百分比 (0~100)
├─ 当前节点名称
├─ 预计剩余节点数
└─ 预览图像 (二进制帧, BinaryEventTypes.PREVIEW_IMAGE)
```

预览图像使用轻量级的 TAESD（Tiny AutoEncoder for Stable Diffusion）实时解码 latent → image，在浏览器中即时显示生成过程。

---

## 十、设计评价

### 亮点

1. **缓存层次丰富:** 从简单的 ID 缓存到输入哈希缓存再到内存感知缓存，覆盖了从开发调试到生产部署的全场景
2. **动态图执行力:** 支持运行时节点生成，为条件逻辑、循环、动态调度提供了基础
3. **类型安全:** 节点输入输出类型在运行时验证，配合组合类型和通配符，既严格又灵活
4. **非侵入性:** 节点的异步特性（IS_CHANGED, VALIDATE_INPUTS）通过约定方法实现，不改变核心执行逻辑

### 潜在改进

1. **缓存序列化:** 当前缓存全部在内存中，跨会话不持久化。长时间运行大量工作流时内存压力较大
2. **并行执行:** 引擎严格按拓扑序串行执行节点，未利用 DAG 中无依赖节点的并行能力
3. **动态图回溯:** 动态生成的节点在出错时，错误追踪和恢复路径不够清晰
4. **缓存粒度:** 节点级缓存粒度较粗，对于内部有多个子步骤的复合节点无法复用部分计算结果

### 可借鉴的设计

1. **分层缓存抽象:** 定义 Cache → KeySet → Eviction 接口，策略可插拔组合，适用于任何有依赖关系的计算系统
2. **层级失效传播:** HierarchicalCache 的模式可应用于构建系统、CI/CD 缓存、编译缓存等场景
3. **动态图与静态图共存:** DynamicPrompt 模式让计算框架可以灵活扩展而不改变用户提交的原始图结构
