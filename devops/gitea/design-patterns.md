# Gitea 设计模式与工程亮点

> **关联文档:** [Gitea 主解读](./README.md) · [系统架构设计详解](./system-arch.md) · [Actions CI/CD 实现分析](./actions.md)

---

Gitea 作为一个经过近 10 年演进的 Go 项目，代码库中沉淀了大量优秀的设计模式和工程实践。本篇聚焦其中最值得学习的 6 个设计。

## 1. 事件通知总线（Observer 模式）

`services/notify/` 是 Gitea 最优雅的设计之一。它实现了**事件驱动的观察者模式**，使核心业务逻辑不关心"通知谁"。

```go
// services/notify/notify.go
var notifiers []Notifier

func RegisterNotifier(notifier Notifier) {
    go notifier.Run()
    notifiers = append(notifiers, notifier)
}

func NewIssue(ctx context.Context, issue *issues_model.Issue, mentions []*user_model.User) {
    for _, notifier := range notifiers {
        notifier.NewIssue(ctx, issue, mentions)
    }
}
```

**设计意图：** 当 Issue 创建、PR 合并等事件发生时，需要触发多个后续动作——发送 Webhook、发邮件、更新搜索索引、触发 Actions 流水线。如果这些逻辑散落在业务代码中，每次新增通知渠道都要修改核心流程。

**解决方式：** 定义 `Notifier` 接口，所有通知机制（Webhook、Mailer、Indexer、Actions 触发器、Mirror 更新）都注册为该接口的实现。`RegisterNotifier()` 在初始化时调用，业务代码只需调用通知总线的统一方法。

```
核心业务代码
    │
    ▼
services/notify/  ←── register ── WebhookNotifier   → services/webhook/
                      register ── MailNotifier       → services/mailer/
                      register ── IndexerNotifier    → services/indexer/
                      register ── ActionsNotifier    → services/actions/
                      register ── MirrorNotifier     → services/mirror/
```

**效果：** 新增通知渠道只需实现 `Notifier` 接口并调用一次 `RegisterNotifier()`，零侵入核心业务代码。

## 2. 泛型队列系统

`modules/queue/` 是 Go 泛型在开源项目中的优秀实践。它实现了一个类型安全的泛型工作池队列，支持三种后端存储。

核心接口设计：

```go
// modules/queue/queue.go
type WorkerPoolQueue[T any] struct {
    baseQueue       baseQueue[T]     // 底层存储抽象
    workerNum       int              // 并发 Worker 数
    workerMaxNum    int              // 最大 Worker 数（自动扩缩）
    handler         HandlerFunc[T]   // 处理函数
    // ...
}
```

**三种后端实现：**

| 后端 | 存储方式 | 持久化 | 适用场景 |
|------|----------|--------|----------|
| Channel | 内存 channel | ❌ | 开发/单实例 |
| LevelDB | 本地文件 | ✅ | 单实例生产 |
| Redis | Redis List | ✅ | 多实例集群 |

**设计特点：**

- **泛型安全**：`WorkerPoolQueue[T any]` 消除了类型断言，编译期保证类型正确
- **自动扩缩**：根据队列积压情况动态调整 Worker 数量
- **统一管理**：`Manager` 集中管理所有队列，提供 `Add()` 和 `Remove()` 方法
- **优雅关闭**：支持 `Shutdown()` 等待所有任务完成

```go
// 使用示例
type WebhookTask struct {
    ID     int64
    Payload []byte
}

queue, _ := queue.CreateQueue("webhook", webhookHandler, queue.Config{
    Type:    "redis",
    Length:  100,
    WorkerNum: 5,
})
queue.Push(WebhookTask{ID: 1, Payload: data})
```

## 3. 策略模式：文件存储抽象层

`modules/storage/storage.go` 定义了一个统一的 `ObjectStorage` 接口，业务代码不需要知道文件存哪里。

```go
type ObjectStorage interface {
    Open(path string) (Object, error)
    Save(path string, r io.Reader, size int64) (int64, error)
    Stat(path string) (os.FileInfo, error)
    Delete(path string) error
    ServeDirectURL(...) (*url.URL, error)
    IterateObjects(basePath string, iterator func(string, Object) error) error
}
```

**已实现的策略：**

| 后端 | 类型 | 适用场景 |
|------|------|----------|
| Local | 本地文件系统 | 单机部署 |
| MinIO/S3 | 对象存储 | 分布式部署 |
| Azure Blob | 云存储 | Azure 环境 |

**使用方式：** 不同用途预定义了全局实例，初始化时按配置创建：

```go
var (
    Attachments      ObjectStorage
    LFS              ObjectStorage
    Avatars          ObjectStorage
    Packages         ObjectStorage
    Actions          ObjectStorage
    ActionsArtifacts ObjectStorage
)
```

**工程价值：** 将"存储"抽象为一个接口，使得 Gitea 可以适配任意存储后端而不影响业务代码。新存储类型只需实现 `ObjectStorage` 接口并通过 `RegisterStorageType()` 注册。

## 4. 插件式认证框架

`services/auth/` 定义了可组合的认证系统，支持多种认证方式的自由组合。

核心接口：

```go
// services/auth/auth.go
type Method interface {
    Auth(ctx *context.Context) (*user_model.User, error)
}

type PasswordAuthenticator interface {
    Authenticate(ctx context.Context, user *user_model.User, login, password string) (*user_model.User, error)
}
```

**内置认证方法：**

| 方法 | 实现位置 | 使用场景 |
|------|----------|----------|
| Session | `services/auth/session.go` | Web 页面（基于 Cookie） |
| Basic Auth | `services/auth/basic.go` | API 调用（HTTP Basic） |
| OAuth2 | `services/auth/oauth2.go` | 第三方登录（GitHub/Google 等） |
| ReverseProxy | `services/auth/reverseproxy.go` | 企业反向代理认证 |
| SSPI | `services/auth/sspi.go` | Windows 集成认证 |

认证链通过 `NewGroup()` 组合多个 `Method`：

```go
func NewGroup(methods ...Method) Method {
    return &Group{methods: methods}
}
```

**设计特点：** 每个认证方法独立实现，通过组合模式串联。系统启动时根据配置组合认证链，运行时按序尝试直到认证成功。

## 5. 编译时双实现：Git 操作抽象

`modules/git/` 通过 Go 构建标签实现**编译时的双实现切换**，同时支持原生 Git CLI 和纯 Go 的 `go-git` 实现。

```
modules/git/
├── command.go               [通用接口定义]
├── repo_commit.go           [通用提交操作]
├── ...
├── repo_attribute_gogit.go  [//go:build gogit]
├── repo_attribute_nogogit.go [//go:build !gogit]
├── ...
```

**两个实现对比：**

| 维度 | 原生 Git CLI | go-git |
|------|-------------|--------|
| 依赖 | 系统安装 git | 纯 Go，无外部依赖 |
| 性能 | 较快（C 实现） | 稍慢（Go 实现） |
| 功能覆盖 | 完整 | 基本完整（少数高级功能不全） |
| 构建标签 | `!gogit`（默认） | `gogit` |

**文件命名约定：** `_gogit.go` 和 `_nogogit.go` 后缀清晰标识不同实现分支，`//go:build` 条件编译确保编译时只包含一个分支。

```go
// repo_attribute_gogit.go
//go:build gogit

package git

func (repo *Repository) Attributes(data ...string) (map[string]string, error) {
    // go-git 实现
}

// repo_attribute_nogogit.go
//go:build !gogit

package git

func (repo *Repository) Attributes(data ...string) (map[string]string, error) {
    // 原生 git CLI 实现
}
```

**工程价值：** 在没有系统 git 的环境中（如 Docker 最小镜像），可以使用 go-git 编译版本；在完整生产环境中，使用原生 CLI 获得更好性能。零运行时开销，编译时已确定。

## 6. 优雅关闭与零停机重启

`modules/graceful/` 实现了完整的服务器优雅关闭机制，支持无中断重启。

**状态机设计：**

```
Init → Running → ShuttingDown → Terminate
```

**核心机制：**

| 特性 | 实现方式 |
|------|----------|
| 关闭回调 | `RunAtShutdown()` / `RunAtTerminate()` 注册清理任务 |
| Context 传播 | 通过 context.Context 级联传播关闭信号 |
| Socket 继承 | 通过 `SO_REUSEPORT` 或 fork+exec 传递文件描述符 |
| 超时控制 | `HammerContext()` 在超时后强制关闭 |
| 服务管理 | `Manager` 统一管理 HTTP/gRPC/内部服务 |

**使用方式：**

```go
// 注册关闭回调
graceful.RunAtShutdown(func() {
    db.Close()
    queue.GetManager().Shutdown()
})

// 在 context 中传播关闭信号
select {
case <-graceful.GetManager().ShutdownContext().Done():
    // 收到关闭信号，开始清理
case err := <-serverError:
    // 服务错误
}
```

**零停机重启流程：**

1. 父进程监听 SIGHUP 信号
2. 父进程 fork 子进程，通过文件描述符传递监听 socket
3. 子进程启动并接管连接
4. 父进程进入 ShuttingDown 状态，不再接受新连接
5. 父进程等待现有请求完成（超时后强制关闭）
6. 父进程退出

这种方法在自托管 DevOps 平台中极为实用——用户无感知的版本更新。

---

> **GitHub:** [github.com/go-gitea/gitea](https://github.com/go-gitea/gitea) — `services/notify/` | `modules/queue/` | `modules/storage/` | `services/auth/` | `modules/git/` | `modules/graceful/`
