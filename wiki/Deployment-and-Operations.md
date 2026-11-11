# 部署与运维

[← 返回 Wiki 首页](Home.md) · 生产检查清单见
[docs/ops-deploy-checklist.md](../docs/ops-deploy-checklist.md)

## 部署方案

| 方案 | 入口 | 要点 |
|---|---|---|
| 二进制一键安装 | [scripts/install.sh](../scripts/install.sh) | 下载 Release 到 `~/.local/bin`，`imgli serve`；适合单机 |
| 预构建镜像 | `ghcr.io/yixian-huang/imgli:latest` | 挂载 `/data`，设置 `IMGLI_BASE_URL` |
| Compose | [docker-compose.yml](../docker-compose.yml) / [deploy/compose.prod.example.yml](../deploy/compose.prod.example.yml) | 生产示例仅回环 `127.0.0.1:8686`、带 healthcheck、可选 Postgres 16、日志轮转 |
| TLS 反代 | [deploy/Caddyfile.example](../deploy/Caddyfile.example) · [deploy/traefik.labels.example.yml](../deploy/traefik.labels.example.yml) | Caddy 示例限制 `request_body max_size 32MB` |
| systemd | [deploy/imgli.service.example](../deploy/imgli.service.example) | 内存上限与 `GOMEMLIMIT`、`ProtectSystem=strict` + `ReadWritePaths` 放行二进制升级 |
| 源码 / 手工 | [Dockerfile](../Dockerfile) · [Makefile](../Makefile) | 三阶段构建：node:24 → golang:1.26 + vips → alpine 运行时 |

[Dockerfile](../Dockerfile) 运行时以 uid 1000 的 `imgli` 用户执行；
[docker-entrypoint.sh](../docker-entrypoint.sh) 会先修正绑定挂载中
`imgli.db` / `-wal` / `-shm` 的属主再降权。

## 部署前 / 后必查

```bash
# 部署前
imgli doctor -config /path/to/imgli.yaml
# 公网部署：IMGLI_BASE_URL=https://域名、IMGLI_TRUST_PROXY=true

# 部署后 5 分钟内
curl -fsS https://你的域名/healthz
./scripts/ops-smoke-public.sh https://你的域名     # / + /healthz + id="root"
```

生产向清单（配置形状抽检、契约变更护栏、回滚步骤）见
[docs/ops-deploy-checklist.md](../docs/ops-deploy-checklist.md)；
发版 / CI / 冒烟见 [docs/ops-release.md](../docs/ops-release.md)。

## 健康检查与可观测面

| 入口 | 用途 |
|---|---|
| `GET /healthz` | 存活探针；Docker `HEALTHCHECK` 使用 |
| `imgli doctor` | data 目录可写、DB 连通、base_url 形态、trust_proxy、监听地址、local 策略写读删；FAIL → exit 1 |
| `GET /api/v1/admin/system/health` | 同源健康报告（管理后台「系统 / 运维」） |
| `GET /api/v1/admin/system/version` + `check-update` + `upgrade` | 版本展示、探测 GitHub Release、一键二进制升级（校验 checksum；Docker 内禁用，请换镜像） |
| `GET /api/v1/admin/stats`、`/admin/referers/images`、`/admin/logs` | 运营统计、Referer 防盗链、审计日志 |
| [deploy/ops/health-check.sh](../deploy/ops/health-check.sh) | 外部快照：磁盘 / 进程 / 队列 / cgroup 内存 / HTTP，可 `AUTO_RESTART=1` |

日志统一 `log/slog` 结构化输出；GORM logger 为 Warn + 200ms 慢查询。
**没有 Prometheus / OpenTelemetry / metrics 端点。**
统计只覆盖源站 `/i` 门禁，CDN 边缘命中不可见
（[deploy/ops/README.md](../deploy/ops/README.md)、
[deploy/ops/cloudflare-i-img-li-cache-checklist.md](../deploy/ops/cloudflare-i-img-li-cache-checklist.md)）。

## 备份与恢复

[docs/backup.md](../docs/backup.md) 是 SSOT：列出要备份的内容（数据库、本地存储对象、
配置），SQLite 在线 `.backup` 与恢复、Docker 卷注意事项、PostgreSQL 的
`pg_dump` / `pg_restore`，以及恢复后的自检。
[deploy/ops/backup-sqlite.sh](../deploy/ops/backup-sqlite.sh) 提供在线备份 +
14 天保留的实现。

## 例行运维

| 任务 | 入口 |
|---|---|
| 生命周期清理 | 每小时自动执行；管理端 `/admin/cleanup/preview` 预览 + `/admin/cleanup/run` 确认；边界见 [docs/ops-cleanup-cdn-boundary.md](../docs/ops-cleanup-cdn-boundary.md) |
| 用户组保留 / 强制存活 | 管理端用户组配置与 lifecycle preview/apply；见 [docs/user-groups-lifecycle.md](../docs/user-groups-lifecycle.md) |
| 跨策略存储搬迁 | 管理端任务（可续跑/取消）或 CLI；见 [docs/storage-migrate.md](../docs/storage-migrate.md) |
| 版本升级 | 二进制部署可用后台一键升级；Docker 部署换镜像 tag；升级前后按 [docs/ops-deploy-checklist.md](../docs/ops-deploy-checklist.md) 冒烟 |
| 机审旁路 | [deploy/ocr-paddle/](../deploy/ocr-paddle/README.md) 的 OCR 服务与 `ALLOW_IPS`；抽检见 [docs/moderation-spot-check.md](../docs/moderation-spot-check.md) |
| 脚本索引 | [deploy/ops/README.md](../deploy/ops/README.md) · [scripts/](../scripts/) |

## 运维脚本速查（[scripts/](../scripts/)）

| 脚本 | 用途 |
|---|---|
| `install.sh` | 一键安装 Release 二进制 |
| `ops-smoke-install.sh` / `ops-smoke-public.sh` | 全新安装冒烟 / 公网冒烟 |
| `ops-deploy-baili.sh` | 生产机 Release 部署 + systemd 重启 |
| `pre-tag-check.sh` | 打 tag 门禁（CHANGELOG / 干净树 / main CI） |
| `s3-vendor-matrix.sh` / `s3-vendor-e2e.sh` | S3 厂商兼容矩阵与 app 层全链 e2e |
| `webdav-vendor-matrix.sh` | WebDAV 厂商矩阵 |
| `subset-watermark-font.sh` | 重新生成水印中文字体子集 |
| `probe-private-object-anon.sh` | 探测私有对象是否可被匿名 2xx 访问 |
| `loadtest.py` | 标准库实现的读/写压测（见 [scripts/README-loadtest.md](../scripts/README-loadtest.md)） |

相关：[Configuration.md](Configuration.md) · [Troubleshooting.md](Troubleshooting.md) ·
[Storage-and-Media.md](Storage-and-Media.md)
