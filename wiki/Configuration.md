# 配置

[← 返回 Wiki 首页](Home.md)

## 两条配置通道（不要混淆）

| 通道 | 放什么 | 在哪里改 |
|---|---|---|
| **运维层** | 监听地址、公网 Base URL、数据目录、数据库 DSN、反代信任、限速倍率、源站缓存、SSRF 放行 | 默认值 → YAML（`imgli serve -config imgli.yaml`）→ `IMGLI_*` 环境变量 |
| **产品层** | 站名、注册模式、用户组与配额/生命周期、存储策略、SMTP、机审、OIDC、外观主题、公告 | 数据库 `Setting` KV 表 → **管理后台**（`GET/PUT /api/v1/admin/settings`） |

`internal/config/config.go` 只解析运维参数。样例：
[deploy/imgli.example.yaml](../deploy/imgli.example.yaml)。
约 30 个 settings 键由 `internal/model/db.go` 的 `Seed` 幂等播种。

## 优先级

```text
内置默认值  <  YAML 配置文件  <  IMGLI_* 环境变量
```

环境变量总是最后生效，适合容器与 systemd 部署；YAML 适合把一组运维参数版本化。

## 运维层环境变量

| 变量 | 必需 | 默认 | 含义 |
|---|---|---|---|
| `IMGLI_BASE_URL` | **公网部署必需** | `http://localhost:8686` | 浏览器实际访问的源（无尾斜杠）。影响外链、邮件链接、Secure Cookie 与 CSRF Origin 白名单 |
| `IMGLI_TRUST_PROXY` | **有反代时设 true** | `false` | 信任 `X-Forwarded-For`（仅用于可信反代之后），影响限速客户端 IP 归因 |
| `IMGLI_LISTEN` | 可选 | `:8686` | 监听地址 |
| `IMGLI_DATA_DIR` | 可选 | `./data` | 本地存储、SQLite、头像与 `.serve-cache` 目录；镜像内常为 `/data` |
| `IMGLI_DATABASE_DRIVER` | 可选 | `sqlite` | `sqlite` \| `postgres` |
| `IMGLI_DATABASE_DSN` | Postgres 时必需 | SQLite 自动路径 `<data_dir>/imgli.db` | PostgreSQL DSN；SQLite 请留空 |
| `IMGLI_FETCH_ALLOW` | 可选 | 空 | URL 抓取上传额外放行的 host / CIDR；默认拒绝内网目标 |
| `IMGLI_RATE_LIMIT_MULT` | 可选 | `1` | 全局限速倍率；生产保持 1，仅测试环境放大 |
| `IMGLI_SERVE_CACHE_DISABLED` | 可选 | `false` | 关闭公开 `/t` 与未 302 的 `/i` 本地代理缓存 |
| `IMGLI_SERVE_CACHE_MAX_BYTES` | 可选 | `0`（=512MiB） | `{data_dir}/.serve-cache` 容量上限 |

其它运行时变量：

| 变量 | 用途 |
|---|---|
| `IMGLI_TOKEN` / `IMGLI_BASE_URL` | CLI `upload` / `import-dir` 的目标站与凭据 |
| `IMGLI_THUMB_MAX_PIXELS` / `IMGLI_THUMB_MAX_BYTES` / `IMGLI_THUMB_CONCURRENCY` | 缩略图解码安全上限与并发（`internal/imaging/imaging.go`，默认 16MP / 24MiB） |
| `VIPS_CONCURRENCY` | libvips 并发上限，官方镜像默认 `2` |
| `IMGLI_TEST_PG_DSN` | 测试切 PostgreSQL |
| `IMGLI_TEST_S3_*` / `IMGLI_TEST_WEBDAV_*` / `IMGLI_TEST_FTP` | 厂商 live 测试凭据，默认全部 skip |

前端**没有 `.env*` 与 `VITE_*` 变量**，运行时配置来自 `GET /api/v1/config`
（见 [Frontend.md](Frontend.md)）。

## YAML 示例

```yaml
# imgli serve -config imgli.yaml
listen: ":8686"
base_url: "https://img.example.com"
data_dir: "/opt/imgli/data"
database:
  driver: sqlite          # sqlite | postgres
  # dsn: "host=127.0.0.1 user=imgli password=... dbname=imgli port=5432 sslmode=disable"
trust_proxy: true
# serve_cache_disabled: false
# serve_cache_max_bytes: 0   # 0 = 512MiB
# fetch_allow:
#   - "10.0.0.0/8"
```

## 数据库

- 默认 SQLite：纯 Go 驱动（`glebarez/sqlite` → `modernc.org/sqlite`），无 CGO；
  pragma 含 WAL、`busy_timeout(5000)`、`foreign_keys(1)`、`mmap_size(0)`、
  `cache_size(-8000)`、`temp_store(FILE)`，WAL 不可用时回退 DELETE。
- PostgreSQL：`IMGLI_DATABASE_DRIVER=postgres` + `IMGLI_DATABASE_DSN`；
  [deploy/compose.prod.example.yml](../deploy/compose.prod.example.yml) 有可选服务示例。
- 迁移随 `serve` 自动执行，也可 `imgli migrate` 手动执行；无 SQL 迁移文件，
  详见 [Architecture.md](Architecture.md#数据模型)。
- 绑定挂载不要放在 NFS / CIFS 上放 SQLite 文件；见
  [docker-compose.yml](../docker-compose.yml) 注释与 [docs/backup.md](../docs/backup.md)。

## 改配置后的自检

```bash
imgli doctor                              # FAIL → exit 1；仅 WARN → exit 0
imgli doctor -config /path/to/imgli.yaml  # 用指定配置文件自检
curl -fsS http://localhost:8686/healthz   # 存活探针
```

`doctor` 检查 data 目录可写、数据库连通、`base_url` 形态、`trust_proxy` 提示、
监听地址，并对已启用的 local 存储策略做写/读/删探针。
管理后台「系统 / 运维」提供同源报告：`GET /api/v1/admin/system/health`。

相关：[Deployment-and-Operations.md](Deployment-and-Operations.md) ·
[Troubleshooting.md](Troubleshooting.md) ·
[docs/security-hardening.md](../docs/security-hardening.md)
