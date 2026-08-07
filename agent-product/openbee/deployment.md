# OpenBee 分发与部署

> 本解读按 agent 类规格产出，配套 [README.md](README.md)。本文梳理 OpenBee 的多渠道分发与部署形态：跨平台二进制、npm 平台子包、系统服务自启、守护进程化与自升级。

## 部署形态概览

OpenBee 是**单二进制 + 本地 SQLite** 的自托管服务，部署目标极简：能跑一个原生的 `openbee` 二进制即可，无数据库服务、无外部依赖。分发覆盖四个渠道，对应不同用户心智：

| 渠道 | 用户 | 技术 |
|------|------|------|
| npm `@theopenbee/cli` | Node 开发者 | optionalDependencies 平台子包 + Node 引导脚本 |
| Homebrew / Scoop | macOS / Windows 用户 | GoReleaser 自动发布 tap/bucket |
| 脚本安装 | 通用 | `install.sh` / `install.zh.sh`（SHA256 校验） |
| GitHub Releases | 手动 | tar.gz / zip + deb / rpm |

## 跨平台二进制：GoReleaser 构建矩阵

`.goreleaser.yml` 用 `goos: linux/darwin/windows` × `goarch: amd64/arm64` 构建 6 个组合，`CGO_ENABLED=0` 保证静态链接、跨平台分发无需 C 依赖（[.goreleaser.yml:8-8]()）。归档按 `openbee-v{version}-{os}-{arch}` 命名，Windows 用 zip（[.goreleaser.yml:33-33]()）。

<!-- snippet: builds: -->

发布自动产出 `checksums.txt`（sha256）供安装脚本与自升级校验（[.goreleaser.yml:38-38]()）。

<!-- snippet: checksum: -->

并推送 Homebrew cask（到 `theopenbee/homebrew-tap`，[.goreleaser.yml:59-59]()）与 Scoop（`theopenbee/scoop-bucket`）。nfpm 统一出 **deb + rpm**，`bindir: /usr/bin`，面向 Debian/RHEL 系直接 yum/apt 装（[.goreleaser.yml:77-77]()）。

<!-- snippet: homebrew_casks: -->
<!-- snippet: nfpms: -->

## npm：optionalDependencies 平台子包，而非下载脚本

npm 渠道的妙处是**没有 postinstall 下载脚本**，而是借用 npm 自身的平台解析：主包 `@theopenbee/cli` 声明 6 个 `optionalDependencies` 平台子包（`cli-linux-x64`、`cli-darwin-arm64` 等，[npm/packages/cli/package.json:14-14]()），每个子包的 `package.json` 用 `os`/`cpu` 字段限定宿主（如 `os:["linux"], cpu:["x64"]`）。npm 装包时按当前平台环境自动挑装对应子包——**平台二进制由 npm 的依赖解析隐式完成**。

<!-- snippet:   "optionalDependencies": { -->

`bin/openbee` 是一个 Node 引导脚本（[npm/packages/cli/package.json:6-6]()），运行时映射 `process.platform-arch` 到子包，`require.resolve` 定位包根，再 `spawnSync` 交出进程（stdio inherit），把参数透传给原生二进制。这样 Node 只是"薄壳"，重量级的 Go 进程收到完整的参数。

<!-- snippet:   "bin": { -->
<!-- snippet:     "openbee": "./bin/openbee" -->

## 系统服务自启：build tags 分平台 Manager

`openbee service install/uninstall/start/stop/status` 让日常免手管进程。核心是通过 **build tags** 编译出平台对应的 `Manager` 实现（`manager_darwin.go` / `manager_linux.go` / `manager_other.go`），接口统一为 `Install/Uninstall/Start/Stop/Status`（[cmd/openbee/internal/cli/servicecmd/manager.go:87-93]()）。

- **macOS（launchd）**：生成 `~/Library/LaunchAgents/com.theopenbee.openbee.plist`，`RunAtLoad` + `KeepAlive`（崩溃自动重启）、`ThrottleInterval` 防频繁重启，启动命令 `openbee server -c <config>`。安装走 `launchctl bootstrap`（[cmd/openbee/internal/cli/servicecmd/manager_darwin.go:42-48]()）。
- **Linux（systemd）**：生成 `/etc/systemd/system/openbee.service`，`Type=simple`、`Restart=on-failure`，但 `ExecStart` 用 `/bin/bash -ilc 'exec ...'` 包裹——这是为了继承用户登录 shell 的 `PATH`（nvm/conda 里的引擎 CLI），避免 systemd 环境缺少引擎路径。`preflightRoot` 强制 root 权限。

LaunchAgents 的 plist 模板见 `servicecmd/templates/launchd.plist.tmpl`。

<!-- snippet: func (darwinManager) plistPath() (string, error) { -->

## 守护进程化：re-exec + 环境变量标记

`openbee server --daemon` 不做标准 fork，而是**用环境变量 `OPENBEE_DAEMON=1` 区分父/子**：父进程解析自身 exe，用 `Setsid: true`（Unix 分离会话，脱离终端）生成子进程并以 `server -c <cfg>` 重新执行（[cmd/openbee/internal/cli/daemoncmd/process_unix.go:18-37]()），等 100ms 确认子进程存活后再写 PID 文件、父进程退出（[cmd/openbee/internal/cli/daemoncmd/lifecycle.go:106-110]()）。子进程检测到 `isDaemonChild()` 后把 stdout/stderr 重定向到日志文件。

**鲁棒性细节**：写 PID 文件前先 `time.Sleep(100ms)` 并 `IsProcessAlive` 确认子进程没立即崩掉——避免留下"写了 PID 但进程已死"的假象；存在存活 daemon 时拒绝重复启动，清理 stale PID 后再 spawn。

## 数据与日志落盘布局

所有运行态数据集中在 `~/.openbee/`（`OpenbeeHomeDir` 缓存解析），细分目录职责：

| 路径 | 用途 |
|------|------|
| `~/.openbee/openbee.pid` / `openbee.log` | PID 文件 / 守护进程日志 |
| `~/.openbee/bee` / `~/.openbee/worker` | bee / worker 默认工作目录 |
| `~/.openbee/logs` | 执行日志（execution 日志落文件而非 DB） |
| `~/.openbee/.codex/sessions`、`.pi/sessions` | 引擎会话存储 |
| `./data/openbee.db`（默认，可由配置改） | SQLite 数据库 |

路径常量定义见 `internal/infra/config/paths.go` 与 `config.go`；DB 默认相对路径 `./data/openbee.db`（[cmd/openbee/internal/cli/configcmd/command.go:45-49]()），SQLite 连接走 WAL。

## 自升级：下载 → 校验 → 原子替换

`openbee upgrade` 先取最新版本（默认 CDN `https://dl.theopenbee.cn` 的 `latest.txt`，也可查 GitHub API，[cmd/openbee/internal/cli/upgradecmd/command.go:95-114]()），对比 semver 决定是否更新；下载平台包并 SHA256 边下边校验，解包后把新二进制写入目标旁的临时文件 `.openbee-new-*`，最后 `os.Rename` **原子替换**——避免升级中途崩溃留下半成品二进制。为自托管单二进制产品提供了"一条命令自我更新"的闭环。

## 全局关联

部署形态是 [README.md](README.md)"自托管"定位的落点：单二进制 + 本地目录 + 多渠道分发，全部服务"数字员工平台可以小团队零成本私有化运行"的产品叙事。它与 [data-model.md](data-model.md) 的 SQLite 单连接/WAL 一脉相承（都是去外部依赖），也与 [runtime.md](runtime.md) 把外部 CLI 当引擎依赖的多引擎设计呼应——正因为引擎是外部 CLI，systemd 才需要用 bash -ilc 继承 PATH。
