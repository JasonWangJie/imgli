# 项目概览

[← 返回 Wiki 首页](Home.md)

## 定位

**imgli** 是用 Go 编写的自托管图床：单个二进制内嵌 React 前端，截图上传后直接得到
可复制的链接。公共实例为 [img.li](https://img.li)，产品站为
[imgli.com](https://imgli.com)。功能全表与演示截图见
[README.zh-CN.md](../README.zh-CN.md#特性)。

设计取向：像运维一个小型 Go 服务那样运维图床——默认 SQLite、无外部依赖、
`imgli doctor` 自检、管理后台承载产品级配置；需要时再接入 PostgreSQL、
S3 兼容对象存储与 CDN。

## 能力地图（到哪里找细节）

| 能力 | 一句话说明 | 详细页面 / SSOT |
|---|---|---|
| 单二进制交付 | Go 服务 + `go:embed` 前端，默认纯 Go SQLite 驱动，无 CGO | [Architecture.md](Architecture.md) |
| 多种存储 | local / S3 兼容 / WebDAV / 受限 FTP；策略级 CDN 302、私图预签名 | [Storage-and-Media.md](Storage-and-Media.md) · [S3 矩阵](../docs/s3-compatibility.md) · [WebDAV 矩阵](../docs/webdav-compatibility.md) · [FTP 双轨](../docs/storage-ftp.md) |
| 上传与变换 | 拖拽 / 粘贴 / URL 抓取；受控缩略图宽度；水印、EXIF 剥离、可选原图 WebP | [API-and-CLI.md](API-and-CLI.md) · [Storage-and-Media.md](Storage-and-Media.md) |
| 账号与分享 | 用户组配额/限速/生命周期、游客上传、邀请码、相册、公开访客页、回收站、口令、阅后即焚 | [Authentication-and-Security.md](Authentication-and-Security.md) · [用户组生命周期](../docs/user-groups-lifecycle.md) |
| 内容安全 | 可插拔 NSFW scorer + 外置 OCR 词表、审核队列、拒审邮件 | [Storage-and-Media.md](Storage-and-Media.md) · [机审抽检](../docs/moderation-spot-check.md) |
| 生态对接 | API Token、CLI、OIDC、Webhook、PicGo / ShareX / uPic | [API-and-CLI.md](API-and-CLI.md) · [docs/integrations/](../docs/integrations/README.md) |
| 运维控制台 | doctor 同源健康报告、版本探测与一键二进制升级、清理预览、跨策略搬迁 | [Deployment-and-Operations.md](Deployment-and-Operations.md) · [搬迁文档](../docs/storage-migrate.md) |

## 两种构建形态（重要）

| 形态 | 获取方式 | 图片能力 |
|---|---|---|
| **纯 Go**（CGO 关） | GitHub Release 归档、`make build`、`.goreleaser.yaml` 产物 | JPEG 缩略图；HEIC/HEIF 上传返回 `415 heic_unsupported`；不提供原图转 WebP |
| **libvips**（CGO 开，`-tags vips`） | 官方 `ghcr.io/yixian-huang/imgli` 镜像、`make build-vips` | WebP 缩略图、可选原图转 WebP、HEIC/HEIF 解码为 JPEG 后走现有管线 |

Docker 运行时默认带 `vips`、`libheif`，并用 `VIPS_CONCURRENCY=2` 控制并发；
`imgli doctor` 在具备 HEIC 解码能力但用户组白名单缺少 `heic/heif` 时会给出 WARN。

## 默认行为速查

| 项 | 默认值 / 行为 | 依据 |
|---|---|---|
| 监听 | `:8686` | `internal/config/config.go` |
| 数据目录 | `./data`（镜像内 `/data`），SQLite 为 `<data_dir>/imgli.db` | `internal/config/config.go` |
| 数据库 | SQLite（纯 Go `glebarez/sqlite`），可切 PostgreSQL | `internal/model/db.go` |
| 首个用户 | 第一个注册用户即管理员 | [README.zh-CN.md](../README.zh-CN.md#快速开始) |
| 默认组 Seed | 容量 10GB、单文件 20MB、月流量 5GiB、限速 20/200/1000（分/时/日） | `internal/model/db.go` |
| 游客组 Seed | 单文件 ≤5MB、3 张/日、1d/7d 生命周期 | `internal/model/db.go` |
| URL 抓取 | 默认拒绝内网目标，需 `IMGLI_FETCH_ALLOW` 显式放行 | `internal/handler/ssrf.go` |
| 缩略图宽度 | 白名单 `120,200,240,400,480,800,960,1600` | `internal/handler/thumbwidth.go` |
| 源站缓存 | `{data_dir}/.serve-cache`，默认上限 512MiB，可关 | `internal/servecache/cache.go` |

## 版本与许可

- 产品版本只来自 **git tag**（`vMAJOR.MINOR.PATCH`），构建时用 `-ldflags` 注入
  `internal/version.Version`；不要在 `go.mod` 或 `web/package.json` 里维护版本。
  数据库 `SchemaVersion` 与产品版本号相互独立。
- 许可为 **AGPL-3.0-only**；修改后经网络提供服务需按 AGPL 提供对应源码。闭源商用 /
  多租户 SaaS 见 [COMMERCIAL.md](../COMMERCIAL.md)。历史标签 v0.1.0 / v0.1.1 为 MIT。
- 内嵌 Noto Sans SC 子集遵循 SIL OFL，第三方说明见 [NOTICE](../NOTICE)。

相关：[Architecture.md](Architecture.md) · [Getting-Started.md](Getting-Started.md) ·
[CHANGELOG.md](../CHANGELOG.md) · [ROADMAP.md](../ROADMAP.md)（公开镜像，执行面在 GitHub Issues）
