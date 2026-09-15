---
type: 变更记录
title: MaxKey 解读变更记录
description: MaxKey 项目解读的跨版本变更汇总，按版本倒序，记录每个解读基线的依据 ref 与主要变更内容。
tags: [maxkey, changelog, release-notes]
---

# 变更记录

> 本解读对 MaxKey 的跨版本变更汇总；按版本倒序。上游在 `ReleaseNotes.txt` 维护逐版本条目，本文件只记录「解读内容」自身的基线变化与结论修订，不重复上游发布说明。

## [4.1.12]

依据：`4.1.12`@`3a3a5c765`

本次为**全量重写**，替换并删除了此前的 `README.md`（基线 v4.1.11）与 `login-logout-flow.md` 两篇旧文档。旧文无 frontmatter、使用「第 N 部分」式编号标题，且几乎不含可校验的源码引用；新文档集统一补齐了 frontmatter、改为语义式标题，并为每条论断附上经程序重定位校验的真实 `文件:行号` 引用。

文档集（`auth/maxkey/`）：

- `README.md` —— 总览：定位、规模、技术栈、子文档索引、核心模块导读表、整体架构、亮点与问题、同类对比、解读类型说明、阅读路线。
- `architecture.md` —— 总体架构与设计哲学：模块边界、可部署应用职责、单入口分发、认证与授权分离、设计约定与权衡。
- `protocol-gateway.md` —— 协议适配层：协议分发、OAuth2/OIDC/SAML2/CAS/JWT 适配结构、签名与票据校验策略。
- `authentication.md` —— 认证核心：Provider 选路、登录成功后的状态收敛、多因素建模、trusted 通道风险、两个 provider 模块的复制式分叉。
- `session-token.md` —— 会话与令牌：容器会话与会话有效期的落差、JWT 作为会话句柄、两级会话存储、单点登出与记住我。
- `identity-provisioning.md` —— 身份供给与同步：同步器实现结构、LDAP/AD 与国内办公套件来源、SCIM 2.0 入站供给。
- `multi-tenancy-rbac.md` —— 多租户与权限：`instId` 行级隔离的读写不对称、两套 RBAC 的生效差异、门禁粒度与越权面。
- `data-model.md` —— 数据模型：44 张表的实体关系、核心表字段、存储选型、Schema 演进、读写与缓存路径。
- `deployment.md` —— 部署与运维：端口与上下文路径、容器编排覆盖范围、默认凭据与密钥、备份升级。
- `frontend-and-infrastructure.md` —— 前后端分层与运行时基础设施：依赖方向、装配机制、starter 裁剪、国密覆盖度、前端令牌存储、网关状态。

基线相关的关键事实：

- **4.1.12 无数据库结构变化**：`sql/` 目录无 `v4.1.12`，`git diff 4.1.11 4.1.12 -- sql/` 为空，故数据模型结论沿用 `v4.1.11` 建库脚本。
- **`maxkey-starter-properties` 在 4.1.12 不存在**：引入该模块的提交并非 `3a3a5c765` 的祖先；本基线的 starter 恰好 8 个，通用配置仍由三份 profile 文件各自维护。
- 上游 4.1.12 的主要改动为：调整 Jar 打包方式、应用明细读取改名并加缓存、OAuth 2 授权页跳过审批、刷新令牌逻辑优化、ID 生成器优化、移除外部 SAML 元数据请求。

## 历史

- `v4.1.11` —— 旧版 `README.md` 与 `login-logout-flow.md` 的解读基线；两篇已在 4.1.12 重写时删除。
