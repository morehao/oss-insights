# Starred vs Starred-Go 详细对比分析

> 分析日期: 2026-06-30
>
> Python 版: [maguowei/starred](https://github.com/maguowei/starred) v4.3.0
>
> Go 版: [juev/starred](https://github.com/juev/starred) (latest)

## 一、一句话定性

| 维度 | Python 版 (maguowei/starred) | Go 版 (juev/starred) |
|------|-----------------------------|-----------------------|
| **定位** | 原创、官方实现 | 社区移植、Go 语言重写 |
| **核心理念** | 简洁至上，220 行搞定 | 工程化完善，功能增强 |
| **目标用户** | Python 生态开发者 | Go 生态开发者 |

## 二、功能对比

| 功能 | Python 版 | Go 版 | 差异分析 |
|------|-----------|-------|----------|
| 按语言分类 | ✅ 默认 | ✅ 默认 | 功能一致 |
| 按 Topic 分类 | ✅ `--topic` | ❌ 不支持 | Python 版独有 |
| Topic 热度过滤 | ✅ `--topic_limit` | ❌ 不支持 | Python 版独有 |
| 排序 | ✅ `--sort` | ✅ `-s` | 功能一致 |
| 输出到 stdout | ✅ 默认 | ✅ 默认 | 功能一致 |
| 写入 GitHub 仓库 | ✅ `--repository` | ✅ `-r` | 功能一致 |
| 私有仓库过滤 | ✅ `--private` | ❌ 不支持 | Python 版独有 |
| 自定义模板 | ❌ 不支持 | ✅ `-T` | Go 版独有 |
| 版本信息 | ✅ `--version` | ✅ `-v` | 功能一致 |
| Token 来源 | 环境变量 / 参数 | 环境变量 / 参数 | 功能一致 |
| Username 来源 | 环境变量 USER / 参数 | 仅参数 | Python 版支持环境变量 |

**结论：Python 版功能更全（Topic 分类 + 私有仓库），Go 版有独有优势（自定义模板）。**

## 三、技术架构对比

| 维度 | Python 版 | Go 版 |
|------|-----------|-------|
| **语言** | Python 3.10+ | Go 1.25+ |
| **行数** | ~220 行（3 文件） | ~420 行（3 文件 + 模板） |
| **CLI 框架** | Click | pflag |
| **API 协议** | GraphQL v4 (gql) | REST API v3 (go-github) |
| **HTTP 客户端** | aiohttp (异步) | net/http (标准库) |
| **缓存** | 无 | HTTP 缓存 (gregjones/httpcache) |
| **并发** | 无（串行递归） | ✅ goroutine pool |
| **输出模板** | Python 字符串拼接 | Go text/template |
| **构建发布** | hatchling → PyPI | GoReleaser → GitHub Release + Docker |
| **测试** | 无单元测试 | ✅ 有单元测试 (httptest mock) |
| **Docker** | ❌ 不支持 | ✅ 支持 |

## 四、API 协议差异（核心架构差异）

这是两个项目最根本的设计分歧：

### Python 版 — GraphQL

```
[Python] → gql (GraphQL) → api.github.com/graphql
         → github3.py (REST) → api.github.com/repos (仅写入)
```

- **优势:** 单次查询可获取仓库全部字段（语言、topic、star 数、fork 数等），无需多次 HTTP 请求
- **劣势:** GraphQL schema 需在运行时拉取（`fetch_schema_from_transport=True`），增加冷启动延迟
- **分页:** 串行递归，N 页需 N 次请求，耗时 O(N)

### Go 版 — REST

```
[Go] → go-github (REST) → api.github.com/users/{user}/starred
```

- **优势:** REST API 天然分页（`response.LastPage`），易实现并发拉取
- **劣势:** 无法获取 topic 信息（REST stars 端点不返回 repositoryTopics），需额外请求
- **分页:** goroutine pool 并发拉取，先取第 1 页获取 `LastPage`，然后并发拉取剩余页面

### 并发实现对比

```python
# Python 版 — 串行递归
if has_next:
    items.extend(self.get_user_starred_by_username(username, end_cursor))
```

```go
// Go 版 — goroutine pool (最大 90 并发)
p := pool.NewWithResults[[]*github.StarredRepository]().
    WithMaxGoroutines(90).
    WithContext(ctx).
    WithCancelOnError()
for i := 2; i <= resp.LastPage; i++ {
    page := i
    p.Go(func(ctx context.Context) ([]*github.StarredRepository, error) {
        return g.getStarredRepositories(ctx, username, opt(page))
    })
}
githubRepos, err := p.Wait()
```

Go 版使用了 [sourcegraph/conc](https://github.com/sourcegraph/conc) 库实现 goroutine pool，最大 90 并发，远超 Python 版的串行递归。对于有 1000+ stars 的用户，Go 版性能优势明显。

## 五、Rate Limiting 对比

| 策略 | Python 版 | Go 版 |
|------|-----------|-------|
| 限流检测 | 无 | ✅ 检查 `resp.Rate.Remaining` |
| 自动等待 | 无 | ✅ `time.Sleep(until Reset)` 后重试 |
| HTTP 缓存 | 无 | ✅ `httpcache.NewMemoryCacheTransport()` |

Go 版在健壮性上明显优于 Python 版。Python 版 GraphQL 请求失败时仅打印红色错误退出，无任何重试或降级。

## 六、输出分发对比

### Python 版

- **默认:** stdout
- **仓库写入模式:** `--repository` 指定仓库名
  - 使用 `github3.py`（REST 客户端）查询/创建仓库
  - **自动创建仓库**（仓库不存在时自动创建）
  - **内容对比后更新**（仅在内容不同时提交）
  - 完成后自动打开浏览器

### Go 版

- **默认:** stdout
- **仓库写入模式:** `-r` 指定仓库名
  - 使用 `go-github` REST 客户端
  - **仅更新已有仓库**（仓库不存在时报错退出，不自动创建）
  - **直接写入/更新文件**（无内容对比）
  - 不自动打开浏览器

Go 版仓库写入功能更为保守，不做自动创建仓库的操作。

## 七、代码组织与架构风格

### Python 版 — 极简风格

```
starred/
├── __init__.py    # 1 行版本号
├── starred.py     # 136 行，CLI + 业务 + 渲染全在一个函数
└── githubgql.py   # 86 行，GraphQL 客户端 + 数据模型
```

**特点:** 极度简洁，所有逻辑集中。适合快速理解和修改。

### Go 版 — 工程化风格

```
starred-go/
├── main.go          # 133 行，CLI 入口 + 渲染
├── github.go        # 259 行，GitHub 客户端 + 业务 + 分类 + 写入
├── github_test.go   # 78 行，单元测试
├── templates/
│   └── template.tmpl # 31 行，Go 模板
├── Dockerfile
├── .goreleaser.yaml
└── .github/workflows/
    ├── build.yml
    ├── ci.yml
    └── codeql-analysis.yml
```

**特点:** 工程完备，有测试、Docker、GoReleaser、CodeQL，基础设施完整。

## 八、依赖生态对比

### Python 版依赖（5 个）

| 依赖 | 用途 | 必要性 |
|------|------|--------|
| click | CLI 框架 | 核心 |
| requests | github3.py 间接依赖 | 间接 |
| github3.py | REST API 写入 | 仅 15 行代码使用 |
| gql | GraphQL 客户端 | 核心 |
| aiohttp | gql 传输层 | 核心 (间接) |

**问题:** `github3.py` 仅用于写入仓库场景（约 15 行代码），可通过纯 `requests` + REST API 替代。

### Go 版依赖（5 个直接）

| 依赖 | 用途 | 必要性 |
|------|------|--------|
| go-github/v71 | GitHub REST API 客户端 | 核心 |
| httpcache | HTTP 响应缓存 | 增强 |
| conc | goroutine pool | 核心 (并发) |
| pflag | CLI 解析 | 核心 |
| x/text | 文本大小写处理 | 核心 |

Go 版依赖更聚焦，没有冗余。

## 九、工程完备性对比

| 维度 | Python 版 | Go 版 |
|------|-----------|-------|
| 单元测试 | ❌ 无 | ✅ 有 (httptest mock) |
| CI | pycodestyle + 集成测试 | GoReleaser + Docker build |
| CD / 发布 | PyPI (GitHub Release 触发) | GitHub Release + Docker + GoReleaser |
| Docker | ❌ | ✅ ghcr.io |
| 二进制分发 | ❌ (仅 pip) | ✅ 多平台二进制 |
| 版本嵌入 | Python 常量 | ✅ ldflags 编译时注入 |
| CodeQL | ❌ | ✅ |
| 语言大小写处理 | 无专门处理 | ✅ 内置 60+ 语言名称映射表 |

## 十、综合对比表

| 维度 | Python 版 | Go 版 | 胜出 |
|------|-----------|-------|------|
| **代码简洁度** | ⭐⭐⭐⭐⭐（220 行） | ⭐⭐⭐（420 行） | Python |
| **功能完整度** | ⭐⭐⭐⭐⭐（Topic + 私有仓库） | ⭐⭐⭐⭐（无 Topic） | Python |
| **并发性能** | ⭐⭐（串行） | ⭐⭐⭐⭐⭐（goroutine pool） | Go |
| **健壮性** | ⭐⭐（无重试/限流） | ⭐⭐⭐⭐⭐（限流等待+缓存） | Go |
| **测试覆盖** | ⭐（无） | ⭐⭐⭐⭐（有） | Go |
| **分发方式** | ⭐⭐⭐（pip 安装） | ⭐⭐⭐⭐⭐（二进制+Docker） | Go |
| **自定义能力** | ⭐⭐（固定模板） | ⭐⭐⭐⭐⭐（自定义模板） | Go |
| **可维护性** | ⭐⭐⭐（all-in-one 函数） | ⭐⭐⭐⭐（清晰分层） | Go 略优 |
| **上手难度** | ⭐⭐⭐⭐⭐（极低） | ⭐⭐⭐⭐（低） | Python |
| **依赖管理** | ⭐⭐⭐（github3.py 冗余） | ⭐⭐⭐⭐⭐（干净） | Go |

## 十一、总结

### 选择建议

**选 Python 版当：**
- 你是 Python 开发者
- 需要按 Topic 分类
- 需要包含私有仓库
- 追求极简代码量
- 只需要 stdout 输出到文件

**选 Go 版当：**
- 你是 Go 开发者
- 有大量 stars（>500），需要并发加速
- 需要自定义输出模板
- 需要多平台二进制分发
- 需要 Docker 部署
- 需要更高的 API 限流健壮性

### 两者可互相借鉴的点

- **Python 版可借鉴 Go 版:** 并发拉取、Rate Limiting 处理、单元测试、自定义模板
- **Go 版可借鉴 Python 版:** Topic 分类支持（需额外 GraphQL 或 REST topic endpoint）、自动创建仓库、私有仓库过滤
