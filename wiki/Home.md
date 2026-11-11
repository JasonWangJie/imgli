# imgli 项目 Wiki

本 Wiki 是 **仓库内的工程知识索引**：面向贡献者、二次开发者与自托管运维者，
说明系统如何组成、如何运行、如何验证。它不是产品使用手册的副本。

- 仓库主页与完整特性列表：[README.md](../README.md) · [README.zh-CN.md](../README.zh-CN.md)
- 面向代码变更的说明入口：[readmenew.md](../readmenew.md)
- 逐条 API 端点与错误码参考：[接口文档.md](../接口文档.md)
- 面向最终用户的产品文档：[docs.imgli.com](https://docs.imgli.com)（不在本仓库）
- 工程 SSOT（兼容矩阵、脚本、设计稿、运维清单）：[docs/](../docs/) 与 [docs/documentation-ssot.md](../docs/documentation-ssot.md)

## 版本基线

| 项 | 值 |
|---|---|
| 模块 | `github.com/yixian-huang/imgli` |
| 本文档快照 | 仓库 `CHANGELOG.md` 最新条目 **v0.10.0（2026-09-05）** 之后的代码状态 |
| 版本权威来源 | git tag + `CHANGELOG.md` + 最新 Release（见 [CONTRIBUTING.md](../CONTRIBUTING.md)） |
| Go / Node | `go 1.26.3` / Node ≥ 24 |
| 许可 | AGPL-3.0-only + 商业双许可（[LICENSE](../LICENSE) · [COMMERCIAL.md](../COMMERCIAL.md)） |

> 版本与 API 发生冲突时，以 git tag、`CHANGELOG.md` 与最新 Release 为准。

## 页面导航

| 页面 | 内容 | 主要读者 |
|---|---|---|
| [Overview.md](Overview.md) | 项目定位、能力地图、两种构建形态、许可 | 所有人 |
| [Architecture.md](Architecture.md) | 分层、请求链路、包职责、数据模型、后台任务 | 贡献者 |
| [Getting-Started.md](Getting-Started.md) | 二进制 / Docker / Compose / 源码四种启动方式 | 自托管者 |
| [Configuration.md](Configuration.md) | 配置优先级、环境变量、YAML、settings KV | 运维 |
| [Authentication-and-Security.md](Authentication-and-Security.md) | 会话、Token、OIDC、CSRF/SSRF、限速、权限矩阵 | 贡献者 / 运维 |
| [Storage-and-Media.md](Storage-and-Media.md) | 存储驱动、CDN、缩略图、图片处理、HEIC、搬迁 | 运维 / 贡献者 |
| [API-and-CLI.md](API-and-CLI.md) | HTTP API 分组、上传契约、CLI 子命令、客户端集成 | 集成开发者 |
| [Frontend.md](Frontend.md) | React SPA 结构、路由与权限、上传队列、构建嵌入 | 前端贡献者 |
| [Development-and-Testing.md](Development-and-Testing.md) | 构建目标、测试矩阵、CI、发版流程 | 贡献者 |
| [Deployment-and-Operations.md](Deployment-and-Operations.md) | 部署方案、健康检查、备份、清理、升级 | 运维 |
| [Troubleshooting.md](Troubleshooting.md) | 高频故障与排查顺序 | 运维 |
| [Documentation-Map.md](Documentation-Map.md) | 全仓文档地图与文档同步规范 | 所有人 |

## 30 秒认识 imgli

```text
浏览器 / 客户端
   │  /api/v1/*        业务 API（统一信封、同源校验、会话/Token 鉴权）
   │  /i/*  /t/*       原图 / 缩略图直链（可能 302 到 CDN）
   ▼
Go 单二进制（chi + GORM）
   ├── internal/handler    HTTP 层与中间件
   ├── internal/service   业务服务（上传、审核、存储、统计……）
   ├── internal/model     SQLite / PostgreSQL 数据模型与迁移
   ├── internal/storage   local / s3 / webdav / ftp 驱动
   ├── internal/imaging   纯 Go 或 libvips 图片处理
   └── web/dist           React SPA，go:embed 内嵌
```

关键事实（写文档时最容易弄错的几点）：

1. **没有 JWT、没有 Prometheus/metrics 端点**：会话是数据库里的不透明串 + HttpOnly Cookie，
   API Token 是 `bl_` 前缀的不透明串，库里只存 SHA-256。
2. **两种构建形态**：`make build` / GitHub Release 是纯 Go 归档；官方 Docker 镜像与
   `make build-vips` 带 libvips + libheif。HEIC 与 WebP 编码能力只存在于后者。
3. **业务配置不在 YAML**：站名、注册模式、用户组、存储策略、SMTP、机审等都在
   数据库 `settings` 表、通过管理后台修改；YAML / 环境变量只放运维参数。
4. **迁移不是 SQL 文件**：GORM `AutoMigrate` + `SchemaVersion` v1–v9 幂等数据迁移，
   `serve` 启动时自动执行。
5. **上传不是分片**：浏览器用 XHR 发单个 multipart 请求，只是为了拿进度事件。

## 文档分层与维护约定

依据 [AGENTS.md](../AGENTS.md) 与 [docs/documentation-ssot.md](../docs/documentation-ssot.md)：

| 层 | 位置 | 放什么 |
|---|---|---|
| 工程（Git） | 本仓库 `docs/`、`README*`、`wiki/` | 与代码/版本绑定的矩阵、脚本、清单、架构说明 |
| 产品文档 | 维护者 KB → docs.imgli.com | 终端用户 how-to、FAQ、简短特性说明 |
| 内部 | 维护者 KB | Roadmap、生产运维细节，不进公开仓库 |

**一次职责只在一处编辑**；摘要与链接可以有多份，完整副本不要有多份。
每次代码变更必须在同一任务 / PR 中同步：

1. [`readmenew.md`](../readmenew.md) —— 本次变更对应的说明；
2. 本 Wiki 中受影响的页面 —— 最新行为、用法或维护事项；
3. 项目变更台账 —— 日期、变更内容、影响范围、验证结果（见
   [Documentation-Map.md](Documentation-Map.md) 中的入口状态说明）。

同时仍需按 [CONTRIBUTING.md](../CONTRIBUTING.md) 更新受影响的 `docs/`、`README*` 与
`CHANGELOG.md` 的 `[Unreleased]`。任一项缺失应在交付说明中明确报告为未完成，
不得声称已同步。
