# Gitea 深度解读

> **关联文档:** [系统架构设计详解](./system-arch.md) · [设计模式与工程亮点](./design-patterns.md) · [Actions CI/CD 实现分析](./actions.md) · [API 调用指南](./api-guide.md)

---

## 一、项目概览

Gitea 是一个轻量级的自托管 DevOps 平台，提供 Git 仓库托管、代码审查、CI/CD（Actions）、Package Registry、Wiki、Issue 跟踪等一站式服务。项目始于 2016 年从 Gogs fork，至今已发展成为社区最活跃的自托管 Git 解决方案之一。

与 GitLab 的"全功能但重资源"和 Gogs 的"极简但扩展有限"不同，Gitea 的定位是**在功能完整性与资源消耗之间取得最佳平衡**——单个二进制文件即可运行，支持 SQLite，适合从个人开发者到中小型团队的场景。

| 维度 | 说明 |
|------|------|
| **原始 Fork** | Gogs (2016.11)，至今独立演进 |
| **开源协议** | MIT |
| **开发语言** | Go（后端）+ Vue 3 + TypeScript（前端） |
| **数据库** | SQLite / PostgreSQL / MySQL / MSSQL |
| **构建产物** | 单二进制文件（~100MB），交叉编译支持 |
| **官方仓库** | [github.com/go-gitea/gitea](https://github.com/go-gitea/gitea) |
| **社区活跃度** | ~3,000+ stars, ~500+ 贡献者, 月度 release |

## 二、同类定位

| 项目 | Go 实现 | 单二进制 | 内置 CI/CD | 资源占用 | 适合场景 |
|------|---------|---------|-----------|---------|---------|
| **Gitea** | ✅ | ✅ | ✅ (Actions) | 低 | 个人/小团队 |
| **GitLab** | ❌ (Ruby) | ❌ | ✅ (CI/CD) | 高 | 中大型企业 |
| **Gogs** | ✅ | ✅ | ❌ | 极低 | 极简 Git 托管 |
| **OneDev** | ✅ | ✅ | ✅ | 中 | 需要自定义工作流 |
| **Forgejo** | ✅ | ✅ | ✅ | 低 | Gitea 分支（社区治理） |

## 三、技术栈速览

### 后端

| 层次 | 选型 | 版本 |
|------|------|------|
| 语言 | Go | 1.26.4 |
| 路由框架 | go-chi/chi (自研封装) | v5 |
| ORM | xorm | 最新 |
| 模板引擎 | Go html/template | 标准库 |
| 队列系统 | 自研 (modules/queue) | 泛型 |
| 搜索索引 | Bleve / Elasticsearch | |
| 缓存 | go-chi/cache (自研封装) | |
| 定时任务 | gocron | v2 |
| Markdown 渲染 | 自研 (modules/markup) | 支持多种格式 |

### 前端

| 层次 | 选型 | 版本 |
|------|------|------|
| 框架 | Vue 3 (Composition API) | v3.5 |
| 构建 | Vite 8 + Rolldown | v8 |
| CSS | Tailwind CSS 3 + PostCSS | v3.4 |
| 图标 | Primer Octicons | v19 |
| 代码编辑器 | CodeMirror 6 | v6 |
| 测试 | Vitest + Playwright | v4 / v1.60 |
| 包管理器 | pnpm | v11 |

### 构建体系

Makefile 提供关键目标：`backend`、`frontend`、`build`（全量）、`test`、`lint-go`、`lint-js`、`fmt`、`tidy`。构建标签控制特性开关：`sqlite`、`pam`、`gogit`（纯 Go git 实现）、`bindata`（嵌入静态资源）。

## 四、项目规模

| 类别 | 文件数 | 代码行数 |
|------|--------|----------|
| Go 源码 | ~2,955 | ~475,700 |
| 其中测试 | ~943 | ~145,292 |
| 模板 (`.tmpl`) | ~574 | ~95,651 |
| 前端 (`.ts`/`.vue`/`.js`) | ~270 | ~34,880 |

---

> **GitHub:** [github.com/go-gitea/gitea](https://github.com/go-gitea/gitea) — `cmd/` | `routers/` | `services/` | `models/` | `modules/` | `web_src/`
