# 开发与测试

[← 返回 Wiki 首页](Home.md) · 贡献约定见 [CONTRIBUTING.md](../CONTRIBUTING.md)

## 环境

| 项 | 要求 |
|---|---|
| Go | ≥ 1.26（[go.mod](../go.mod) 为 `go 1.26.3`） |
| Node | ≥ 24 |
| 数据库 | 默认纯 Go SQLite（无 CGO）；PostgreSQL 测试需 `IMGLI_TEST_PG_DSN` |
| 可选 | libvips + libheif 开发包（`make build-vips` / `make test-vips`）、fonttools（字体子集） |

本地与 CI 默认是 **CGO-free 纯 Go** 构建；只有官方 Docker 镜像与
`make build-vips` 打开 `-tags vips`（见 [Overview.md](Overview.md)）。

## 常用命令（[Makefile](../Makefile)）

```bash
make build          # web + build-go（内嵌前端的纯 Go 二进制）
make build-vips     # 需本机 libvips；带 WebP / HEIC 能力
make build-go       # 仅 Go 构建（要求 web/dist 已存在）
make run            # build + ./imgli serve
make docker-build   # 与发行镜像一致（含 libvips）
make test           # go vet ./... + go test ./... -count=1
make test-vips      # CGO_ENABLED=1 go test -tags vips ./internal/imaging/ ./internal/service/upload/
make test-web       # cd web && npm test（Vitest）
make font-subset    # 重新生成水印字体子集（需 fonttools）
make pre-tag TAG=vX.Y.Z
make release-snapshot  # 本地 GoReleaser dry-run
make smoke-public      # 公网冒烟
```

版本号来自 `git describe`，通过
`-ldflags "-X github.com/yixian-huang/imgli/internal/version.Version=$(VERSION)"`
注入；**不要把版本写进 `go.mod` 或 `web/package.json`**。

## 测试分布

| 层 | 位置 | 说明 |
|---|---|---|
| Go 单元 / 集成 | 全仓约 **141 个 `*_test.go`** | `internal/service`（~60，adminsvc/moderation/imagesvc/upload 最多）、`internal/handler`（24，含直链门禁与限速）、`internal/server`（13，`api_admin_test.go` 最大）、`internal/storage`（11）、`internal/imaging`（10）、`internal/model`（9）、`internal/mail`（4）、`internal/version`（3）等 |
| 前端单元 | `web/src/**/*.test.ts(x)` | Vitest + jsdom + Testing Library |
| E2E | [web/e2e/](../web/e2e/) 13 个 spec | Playwright，覆盖 admin / guest / hotlink / invite / lang / mail / main / moderation / plaza / processing / pwa / settings / storage |
| 厂商 live 测试 | `internal/storage/*` | 默认 skip；需 `IMGLI_TEST_S3_LIVE=1` / `IMGLI_TEST_WEBDAV_LIVE=1` / `IMGLI_TEST_FTP=1` 及凭据，脚本见 [scripts/](../scripts/) |
| 冒烟 | `scripts/ops-smoke-install.sh`、`ops-smoke-public.sh` | 全新安装启动 + SPA + 注册 + 上传；公网 `/`、`/healthz`、`id="root"` |

```bash
make test                                  # 默认 SQLite
IMGLI_TEST_PG_DSN='...' make test          # PostgreSQL
cd web && npm test                         # Vitest
cd web && npm run e2e:smoke                # 4 个核心 spec
cd web && npm run e2e                      # 全量（会先构建二进制）
```

## CI（[.github/workflows/ci.yml](../.github/workflows/ci.yml)）

| Job | 内容 |
|---|---|
| backend | `postgres:16-alpine` service；SQLite 与 PostgreSQL 两轮 `go test ./... -count=1` |
| web | `npm ci && npm test && npm run build` |
| binary | `make build` + `/healthz` 与 `id="root"` 冒烟 |
| e2e | 每个 PR 跑 `e2e:smoke`；`main` 分支跑全量 e2e |

PR 还需要 DCO sign-off（`git commit -s`），dependabot / github-actions 提交豁免；
用户可见变更要在 [CHANGELOG.md](../CHANGELOG.md) 的 `[Unreleased]` 加一条。

## 发版流程

完整 playbook：[docs/ops-release.md](../docs/ops-release.md)。短版：

1. PR → CI 全绿 → 合并。
2. `[Unreleased]` 移入带日期的 `## [X.Y.Z]`，刷新底部 compare 链接。
3. `./scripts/pre-tag-check.sh vX.Y.Z`（校验 CHANGELOG 段落、干净树、main CI）。
4. `git tag -a vX.Y.Z -m "vX.Y.Z" && git push origin vX.Y.Z`。
5. [release 工作流](../.github/workflows/release.yml)：GoReleaser 产出多平台
   纯 Go 归档；`ghcr.io/yixian-huang/imgli` 产出 `vX.Y.Z`、`X.Y.Z`、`X.Y`、`latest`
   镜像（含 `-` 的预发布不打浮动 tag）。
6. 冒烟：`./scripts/ops-smoke-public.sh https://…`，`imgli version` 应打印 `vX.Y.Z`。

语义化版本：破坏性 API/配置/存储语义 → MAJOR（0.x 阶段可 MINOR）、新功能 → MINOR、
修复/安全补丁 → PATCH。数据库 `SchemaVersion` 独立，需要运维动作的迁移要在
CHANGELOG 里明确写出。

## 每次变更的文档同步

按 [AGENTS.md](../AGENTS.md) 与
[docs/documentation-ssot.md](../docs/documentation-ssot.md)：同一任务 / PR 中必须更新
[readmenew.md](../readmenew.md)、本 Wiki 受影响的页面、项目变更台账，
并更新受影响的 `docs/`、`README*` 与 `CHANGELOG.md`。PR 模板清单见
[.github/pull_request_template.md](../.github/pull_request_template.md)。
交付说明要列出实际更新的文件 / 页面，缺失项要如实报告。

相关：[Documentation-Map.md](Documentation-Map.md) ·
[Deployment-and-Operations.md](Deployment-and-Operations.md)
