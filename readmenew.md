# imgli · readmenew

本文件是按 [AGENTS.md](AGENTS.md) 维护的**变更说明入口**：每次代码变更都在这里
记录「改了什么、影响什么、怎么验证」。

分工：`README.md` / `README.zh-CN.md` 介绍产品与安装；`wiki/` 是工程知识索引；
本文件是变更与文档导航的最短入口，不复刻上述任一文档的全文。

## 项目速览

| 项 | 值 |
|---|---|
| 定位 | Go 编写的自托管图床，单二进制内嵌 React 前端 |
| 模块 / 入口 | `github.com/yixian-huang/imgli` · `cmd/imgli/main.go` |
| 版本基线 | git tag + `CHANGELOG.md` + 最新 Release；仓库当前最新条目为 **v0.10.0（2026-09-05）** |
| 技术栈 | Go 1.26.3（chi + GORM + SQLite/PostgreSQL）· React 19 + Vite 7 + TypeScript · Tailwind CSS v4 |
| 默认端口 / 数据 | `:8686` · `./data`（镜像内 `/data`） |
| 许可 | AGPL-3.0-only + 商业双许可 |

## 最快上手

```bash
# 二进制（Linux / macOS）
curl -fsSL https://raw.githubusercontent.com/yixian-huang/imgli/main/scripts/install.sh | sh
imgli serve                     # → http://localhost:8686，第一个注册用户即管理员

# Docker
docker run --rm -p 8686:8686 -v imgli-data:/data \
  -e IMGLI_BASE_URL=http://localhost:8686 \
  ghcr.io/yixian-huang/imgli:latest

# 源码（Go ≥ 1.26、Node ≥ 24）
make build && ./imgli serve
```

公网部署必须设置 `IMGLI_BASE_URL=https://你的域名`，有反代时加
`IMGLI_TRUST_PROXY=true`；否则登录/注册会 403「跨站请求被拒绝」。
详见 [wiki/Getting-Started.md](wiki/Getting-Started.md) 与
[wiki/Configuration.md](wiki/Configuration.md)。

## 文档导航

**[wiki 首页](wiki/Home.md)** —— 索引、架构与维护约定。常用页面：

| 想做什么 | 页面 |
|---|---|
| 理解系统组成与请求链路 | [wiki/Architecture.md](wiki/Architecture.md) |
| 配置环境变量 / 区分运维层与产品层配置 | [wiki/Configuration.md](wiki/Configuration.md) |
| 查 API 端点、CLI 命令、客户端集成 | [wiki/API-and-CLI.md](wiki/API-and-CLI.md)（逐条端点与错误码全表见 [接口文档.md](接口文档.md)） |
| 查存储驱动、缩略图、HEIC、机审 | [wiki/Storage-and-Media.md](wiki/Storage-and-Media.md) |
| 前端结构、路由权限、上传队列 | [wiki/Frontend.md](wiki/Frontend.md) |
| 构建、测试、CI、发版 | [wiki/Development-and-Testing.md](wiki/Development-and-Testing.md) |
| 部署、备份、健康检查、升级 | [wiki/Deployment-and-Operations.md](wiki/Deployment-and-Operations.md) |
| 报错排查 | [wiki/Troubleshooting.md](wiki/Troubleshooting.md) |
| 全仓文档地图 | [wiki/Documentation-Map.md](wiki/Documentation-Map.md) |

工程 SSOT 仍在 [docs/](docs/)：`s3-compatibility.md`、
`webdav-compatibility.md`、`storage-ftp.md`、`storage-migrate.md`、
`user-groups-lifecycle.md`、`security-hardening.md`、`ops-deploy-checklist.md`、
`ops-release.md`、`backup.md` 等；分层规则见
[docs/documentation-ssot.md](docs/documentation-ssot.md)。

## 开发速查

```bash
make test          # go vet + go test ./...（默认 SQLite；IMGLI_TEST_PG_DSN 切 PostgreSQL）
make test-web      # Vitest
make test-vips     # 需本机 libvips
cd web && npm run e2e:smoke
```

`make build` / GitHub Release 是**纯 Go 归档**（HEIC 返回 `heic_unsupported`）；
官方 Docker 镜像与 `make build-vips` 带 libvips + libheif（WebP 缩略图、HEIC→JPEG）。
业务配置（站名、用户组、存储策略、SMTP、机审等）在管理后台的 `settings` 表，
不写 YAML。更多见 [CONTRIBUTING.md](CONTRIBUTING.md)。

## 变更记录

倒序追加；每次代码变更都要有对应条目（含修复、重构与内部调整）。

| 日期 | 变更内容 | 影响范围 | 验证 |
|---|---|---|---|
| 2026-11-11 | 建立仓库内的两个文档维护入口：新增 `wiki/`（Home + 12 个主题页：概览、架构、快速开始、配置、认证安全、存储媒体、API/CLI、前端、开发测试、部署运维、排障、文档地图）与 `readmenew.md` | **仅文档**，无产品行为、API、配置或数据模型变化；不改变构建与运行方式 | 事实核对了 `go.mod`、`Makefile`、`web/package.json`、`internal/server/server.go`、`cmd/imgli/main.go`、`deploy/imgli.example.yaml`、`AGENTS.md`、`CONTRIBUTING.md`、`docs/documentation-ssot.md`、`CHANGELOG.md` 及 `docs/` 文件清单；逐个校验了 Wiki 内部与仓库相对链接目标。纯文档变更，未运行构建与测试；未在 `CHANGELOG.md` 的 `[Unreleased]` 添加条目（无用户可见产品变化） |

## 维护约定

按 [AGENTS.md](AGENTS.md) 与 [.github/pull_request_template.md](.github/pull_request_template.md)，
**每次代码变更必须在同一任务 / PR 中同步三项**：

1. 本文件（`readmenew.md`）—— 追加变更记录；
2. [wiki/](wiki/Home.md) 中受影响的页面 —— 更新最新行为、用法或维护事项；
3. 项目变更台账 —— 记录日期、变更内容、影响范围、验证结果。

同时更新受影响的 `docs/`、`README*`、`CHANGELOG.md`（见
[CONTRIBUTING.md](CONTRIBUTING.md)），并在交付说明中列出实际更新的文件 / 页面。

> **入口状态**：项目变更台账在本仓库内尚未定位到文件。若维护在维护者 KB 或外部系统，
> 需维护者提供入口与访问权限；按 `AGENTS.md` 第 17 行，无法定位或无权限时必须
> 在交付说明中报告该项未完成，不得声称已同步，也不得用其它文档替代。
> 本次仅建立 `wiki/` 与 `readmenew.md`，未创建替代性台账。
