# Gitea API 调用指南：纯后端场景

> 适用对象：CI/CD 脚本、自动化工具、微服务集成、第三方平台对接等纯后端调用场景。

---

## 一、获取 API Token

后端调用 Gitea API 只需要一个 **Personal Access Token**（个人访问令牌），不依赖浏览器 Cookie 或 Session。

### 方式一：Web UI 手动创建（推荐）

1. 登录 Gitea → 右上角头像 → **设置** → **应用**
2. 在"管理访问令牌"区域，填写 Token 名称
3. 勾选需要的 **Scope 权限**（见下方表格）
4. 点击 **生成令牌** → **立即复制并保存**（关闭页面后不可再次查看）

```
Token 示例: 7c4a8d09ca3762af61e59520943dc26494f8941b
```

### 方式二：API 自动创建（适合初始化脚本）

```bash
# 使用管理员或用户密码创建 token
curl -X POST 'https://gitea.example.com/api/v1/users/{username}/tokens' \
  -u 'username:password' \
  -H 'Content-Type: application/json' \
  -d '{"name":"ci-token","scopes":["read:repository","write:repository"]}'
```

返回：

```json
{
  "id": 1,
  "name": "ci-token",
  "sha1": "7c4a8d09ca3762af61e59520943dc26494f8941b",
  "scopes": ["read:repository", "write:repository"]
}
```

> `sha1` 字段中的 40 字符值即为你的 Token，只在创建时返回一次。

### 常用 Scope 速查

| Scope | 文件只读 | 文件读写 | 完整访问 |
|-------|---------|---------|---------|
| `read:repository` | ✅ | ✅ | ✅ |
| `write:repository` | ✅ | ✅ | ✅ |
| `read:issue` | — | — | ✅ |
| `read:user` | — | — | ✅ |
| `read:admin` | — | — | ✅ |
| `all` | — | — | ✅ |

> 文件浏览/下载只需 `read:repository`；创建/更新/删除文件需要 `write:repository`。

---

## 二、在请求中携带 Token

推荐通过 HTTP Header 传递，按优先级推荐：

```bash
# ★ 方式一：Authorization: token（最推荐）
curl -H 'Authorization: token 7c4a8d09ca3762af61e59520943dc26494f8941b' \
  'https://gitea.example.com/api/v1/repos/owner/repo/contents/README.md'

# ★ 方式二：Authorization: Basic（兼容旧工具）
curl -u 'username:7c4a8d09ca3762af61e59520943dc26494f8941b' \
  'https://gitea.example.com/api/v1/repos/owner/repo/contents/README.md'

# △ 方式三：URL 参数（不推荐，Token 会出现在访问日志和浏览器历史中）
curl 'https://gitea.example.com/api/v1/repos/owner/repo/contents/README.md?token=7c4a8d09ca3762af61e59520943dc26494f8941b'
```

---

## 三、文件操作 API 示例

所有路径中的 `{owner}` 和 `{repo}` 替换为实际的仓库所有者用户名和仓库名。

### 3.1 目录浏览

```bash
# 列出仓库根目录
curl -H 'Authorization: token xxxxx' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/contents?ref=main'

# 列出子目录
curl -H 'Authorization: token xxxxx' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/contents/src/utils?ref=main'
```

返回格式（每个条目）：

```json
[
  {
    "name": "main.go",
    "path": "src/main.go",
    "type": "file",
    "size": 2048,
    "sha": "abc123...",
    "download_url": "https://gitea.example.com/owner/repo/raw/branch/main/src/main.go"
  },
  {
    "name": "utils",
    "path": "src/utils",
    "type": "dir",
    "sha": "def456..."
  }
]
```

### 3.2 预览文件内容

```bash
# 获取文件元数据 + Base64 正文
curl -H 'Authorization: token xxxxx' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/contents/README.md?ref=main'
```

返回：

```json
{
  "name": "README.md",
  "path": "README.md",
  "type": "file",
  "content": "IyBH...base64编码...",
  "encoding": "base64",
  "size": 1024,
  "sha": "abc123...",
  "html_url": "https://gitea.example.com/owner/repo/src/branch/main/README.md",
  "download_url": "https://gitea.example.com/owner/repo/raw/branch/main/README.md"
}
```

### 3.3 下载原始文件

```bash
# 下载文件原始内容（直接输出到 stdout）
curl -H 'Authorization: token xxxxx' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/raw/branch/main/README.md'

# 保存为文件
curl -H 'Authorization: token xxxxx' -O \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/raw/branch/main/config.yaml'

# 支持 tag 和 commit SHA
curl -H 'Authorization: token xxxxx' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/raw/tag/v1.0.0/CHANGELOG.md'
curl -H 'Authorization: token xxxxx' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/raw/commit/a1b2c3d4/README.md'
```

### 3.4 下载归档文件

```bash
# 下载 ZIP 归档
curl -H 'Authorization: token xxxxx' -o project-main.zip \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/archive/main.zip'

# 下载 tar.gz 归档
curl -H 'Authorization: token xxxxx' -o project-main.tar.gz \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/archive/main.tar.gz'
```

### 3.5 创建文件（需 `write:repository`）

```bash
curl -X POST -H 'Authorization: token xxxxx' \
  -H 'Content-Type: application/json' \
  -d '{
    "content": "IyBORVcgRmlsZQoKVGhpcyBpcyBhIG5ldyBmaWxlIGNyZWF0ZWQgdmlhIEFQSQo=",
    "message": "feat: 通过 API 创建新文件",
    "branch": "main",
    "author": {"name": "Bot", "email": "bot@example.com"}
  }' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/contents/docs/new-file.md'
```

> `content` 字段必须为 **Base64 编码**的文件内容。

### 3.6 更新文件（需 `write:repository`）

```bash
curl -X PUT -H 'Authorization: token xxxxx' \
  -H 'Content-Type: application/json' \
  -d '{
    "content": "5piT5pSv55qE5paH5a2X5YaF5a65",
    "message": "docs: 更新配置文件",
    "branch": "main",
    "sha": "abc123..."  ← 在上次获取 contents 时返回的 sha
  }' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/contents/config.yaml'
```

> `sha` 是当前文件的最新 SHA，用于冲突检测（Gitea 在更新和删除时校验此值）。

### 3.7 删除文件（需 `write:repository`）

```bash
curl -X DELETE -H 'Authorization: token xxxxx' \
  -H 'Content-Type: application/json' \
  -d '{
    "sha": "abc123...",
    "message": "chore: 移除过期配置文件",
    "branch": "main"
  }' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/contents/old-config.yaml'
```

---

## 四、其他常用 API 操作

### 4.1 Commit 历史与状态

```bash
# 获取 Commit 列表（支持分页）
curl -H 'Authorization: token xxxxx' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/commits?sha=main&page=1&limit=30'

# 获取单个 Commit
curl -H 'Authorization: token xxxxx' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/git/commits/a1b2c3d4'

# 获取 Commit 状态（CI 运行结果）
curl -H 'Authorization: token xxxxx' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/commits/a1b2c3d4/statuses'

# 设置 Commit 状态
curl -X POST -H 'Authorization: token xxxxx' \
  -H 'Content-Type: application/json' \
  -d '{
    "state": "success",
    "target_url": "https://ci.example.com/build/123",
    "description": "CI 构建通过",
    "context": "ci/jenkins"
  }' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/statuses/a1b2c3d4'
```

### 4.2 分支管理

```bash
# 列出分支
curl -H 'Authorization: token xxxxx' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/branches'

# 创建分支
curl -X POST -H 'Authorization: token xxxxx' \
  -H 'Content-Type: application/json' \
  -d '{
    "new_branch_name": "feature/new-api",
    "old_ref_name": "main"
  }' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/branches'

# 删除分支
curl -X DELETE -H 'Authorization: token xxxxx' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/branches/feature/new-api'
```

### 4.3 Release 管理

```bash
# 获取 Release 列表
curl -H 'Authorization: token xxxxx' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/releases'

# 创建 Release
curl -X POST -H 'Authorization: token xxxxx' \
  -H 'Content-Type: application/json' \
  -d '{
    "tag_name": "v1.0.0",
    "target_commitish": "main",
    "name": "v1.0.0",
    "body": "### 更新内容\n- 新增功能 A\n- 修复 B",
    "draft": false,
    "prerelease": false
  }' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/releases'
```

### 4.4 Webhook 管理

```bash
# 创建仓库 Webhook
curl -X POST -H 'Authorization: token xxxxx' \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "gitea",
    "config": {
      "url": "https://hook.example.com/gitea",
      "content_type": "json",
      "secret": "my-secret"
    },
    "events": ["push", "pull_request"],
    "active": true
  }' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/hooks'
```

### 4.5 搜索与全局查询

```bash
# 搜索仓库
curl -H 'Authorization: token xxxxx' \
  'https://gitea.example.com/api/v1/repos/search?q=gitea&topic=true&page=1&limit=20'

# 搜索用户
curl -H 'Authorization: token xxxxx' \
  'https://gitea.example.com/api/v1/users/search?q=username'

# 全局 Issue 搜索
curl -H 'Authorization: token xxxxx' \
  'https://gitea.example.com/api/v1/repos/issues/search?q=bug&state=open&limit=50'
```

### 4.6 版本与系统信息

```bash
# 获取 Gitea 版本（无需鉴权）
curl 'https://gitea.example.com/api/v1/version'

# 获取系统设置（无需鉴权）
curl 'https://gitea.example.com/api/v1/settings/api'
```

---

## 五、分页处理

所有列表类 API 都支持分页。Gitea 使用 **基于页码** 的分页方式：

| 参数 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| `page` | int | 页码，从 1 开始 | 1 |
| `limit` | int | 每页条目数 | 由后端配置决定 |

响应头中包含了分页元数据：

```
X-Total-Count: 156      ← 总条目数
X-Page: 1               ← 当前页码
X-PerPage: 30           ← 每页条目数
```

分页循环示例（Shell 脚本）：

```bash
page=1
total_pages=999

while [ "$page" -le "$total_pages" ]; do
  echo "=== Page $page ==="

  response=$(curl -s -H 'Authorization: token xxxxx' \
    "https://gitea.example.com/api/v1/repos/issues/search?state=open&page=$page&limit=50")

  total_count=$(echo "$response" | jq -r 'length')
  echo "Items on this page: $total_count"

  [ "$total_count" -eq 0 ] && break

  page=$((page + 1))
done
```

---

## 六、错误处理

Gitea API 返回统一的错误格式：

```json
{
  "message": "找不到仓库",
  "url": "https://gitea.example.com/api/swagger",
  "documentation_url": "https://docs.gitea.com/api/1.20/"
}
```

### 常见 HTTP 状态码

| 状态码 | 含义 | 常见原因 |
|--------|------|---------|
| **200** | 成功 | 文件读取、列表查询 |
| **201** | 创建成功 | 创建文件、创建 Token |
| **204** | 删除成功（无返回体） | 删除文件、删除分支 |
| **401** | 未认证 | Token 缺失、Token 已撤销 |
| **403** | 无权限 | `read:repository` 尝试写操作、访问私有仓库但 Token 是 `public-only` |
| **404** | 资源不存在 | 路径错误、仓库不存在、Token 无权访问私有仓库 |
| **409** | 冲突 | 更新文件时 `sha` 不匹配（文件已被他人修改） |
| **422** | 参数校验失败 | content 不是合法 Base64、分支名不合法 |
| **429** | 请求频率限制 | 短时间大量请求 |

### 幂等操作建议

```bash
# 删除前先检查是否存在
exists=$(curl -s -o /dev/null -w '%{http_code}' \
  -H 'Authorization: token xxxxx' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/contents/old.md')
[ "$exists" = "200" ] && curl -X DELETE -H 'Authorization: token xxxxx' \
  -H 'Content-Type: application/json' \
  -d '{"sha":"...","message":"删除","branch":"main"}' \
  'https://gitea.example.com/api/v1/repos/{owner}/{repo}/contents/old.md'
```

---

## 七、纯后端 vs 浏览器操作对照

| 维度 | 浏览器用户 | 纯后端 API 调用 |
|------|-----------|----------------|
| **认证凭证** | 用户名+密码 → Session Cookie | Personal Access Token |
| **凭证管理** | 无感，Cookie 自动携带 | 需在 `Authorization` Header 中显式传递 |
| **权限校验** | 基于 Session 中的用户角色 | Token Identity + Scope 双重校验 |
| **文件预览路径** | Web 路径 `/owner/repo/src/branch/main/README.md` | API 路径 `.../api/v1/repos/{owner}/{repo}/contents/README.md` |
| **原始文件下载** | Web 路径 `/owner/repo/raw/branch/main/file` | API 路径 `.../api/v1/repos/{owner}/{repo}/raw/branch/main/file` |
| **文件内容编码** | 直接渲染 HTML/高亮 | Base64 编码（`contents`端点）或原始字节（`raw`端点） |
| **文件写入** | 编辑器 UI → 表单提交 | Base64 编码内容 + SHA 冲突检测 |
| **分页** | 滚动自动加载 | 手动处理 `page`/`limit` + 读取响应头 |
| **Token 生命周期** | 不涉及 | 需手动撤销 / 定期轮换（建议 90 天） |

---

> **GitHub:** [github.com/go-gitea/gitea](https://github.com/go-gitea/gitea) — `routers/api/v1/repo/file.go` | `routers/api/v1/repo/download.go` | `models/auth/access_token.go`
