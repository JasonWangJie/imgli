# 快速开始

[← 返回 Wiki 首页](Home.md) · 完整安装说明见 [README.zh-CN.md](../README.zh-CN.md#快速开始)

## 前置条件

| 方式 | 要求 |
|---|---|
| 预构建二进制 | Linux / macOS / Windows；无运行时依赖 |
| Docker | Docker 或兼容运行时 |
| 源码构建 | Go ≥ 1.26（[go.mod](../go.mod) 为 1.26.3）、Node ≥ 24 |
| 数据库 | 默认内嵌 SQLite；可选 PostgreSQL 16+ |

## 方式一：一键安装二进制（Linux / macOS）

```bash
curl -fsSL https://raw.githubusercontent.com/yixian-huang/imgli/main/scripts/install.sh | sh
imgli serve
# → http://localhost:8686（第一个注册用户即管理员）
```

脚本把当前系统的 [GitHub Release](https://github.com/yixian-huang/imgli/releases)
产物装到 `~/.local/bin`，可用 `PREFIX=...` 改目录，也可 `sh -s -- v0.10.0` 固定版本。
Windows 下载 `Windows_*.zip`，或在 Git Bash 中运行脚本。

## 方式二：预构建 Docker 镜像

```bash
docker run --rm -p 8686:8686 -v imgli-data:/data \
  -e IMGLI_BASE_URL=http://localhost:8686 \
  ghcr.io/yixian-huang/imgli:latest
```

固定版本用 `ghcr.io/yixian-huang/imgli:v0.10.0`。镜像默认带 libvips + libheif，
支持 WebP 缩略图与 HEIC 上传；数据目录为 `/data`。

## 方式三：Docker Compose

```bash
git clone https://github.com/yixian-huang/imgli && cd imgli
docker compose up -d
```

生产向示例（仅回环绑定、健康检查、可选 PostgreSQL、日志轮转）见
[deploy/compose.prod.example.yml](../deploy/compose.prod.example.yml)；
TLS 反代片段见 [deploy/Caddyfile.example](../deploy/Caddyfile.example) 与
[deploy/traefik.labels.example.yml](../deploy/traefik.labels.example.yml)。

## 方式四：源码构建

```bash
make build          # = web(npm install && npm run build) + build-go
./imgli version     # ldflags 注入的 git tag，如 v0.10.0
./imgli serve       # → http://localhost:8686
```

前端产物 `web/dist` 通过 `//go:embed all:dist`（[web/embed.go](../web/embed.go)）
编入二进制；未构建前端时，服务会显示一个内置提示页。
需要 libvips 能力时本机安装 vips/libheif 开发包后用 `make build-vips`
（见 [Makefile](../Makefile)）。开发期可在 `web/` 下 `npm run dev`，
Vite 会把 `/api`、`/i/`、`/t/` 代理到 `http://localhost:8686`。

## 首次启动后

1. 浏览器打开 `http://localhost:8686`，**第一个注册用户自动成为管理员**。
2. 在管理后台配置产品级参数：站名、注册模式、用户组配额与生命周期、
   存储策略、SMTP、机审、外观。它们存数据库 `settings` 表，不写 YAML
   （见 [Configuration.md](Configuration.md)）。
3. 公网部署务必设置 `IMGLI_BASE_URL=https://你的域名`，有反代时加
   `IMGLI_TRUST_PROXY=true`；否则登录/注册会 403「跨站请求被拒绝」
   （见 [Troubleshooting.md](Troubleshooting.md)）。
4. 自检：`imgli doctor`（有 FAIL 时 exit 1，只有 WARN 时 exit 0），
   或管理后台「系统 / 运维」的同源健康报告。

## 验证一次上传

```bash
# 先在 设置 → API Token 创建 Token（明文只显示一次）
curl -X POST http://localhost:8686/api/v1/upload \
  -H "Authorization: Bearer <API_TOKEN>" \
  -F file=@shot.png -F visibility=public
# → data.links.{url,markdown,html,bbcode,thumbnail_url}
```

或使用 CLI：`imgli upload shot.png`（需 `IMGLI_BASE_URL` 与 `IMGLI_TOKEN`），
详见 [API-and-CLI.md](API-and-CLI.md)。

## 下一步

- [Configuration.md](Configuration.md)：配置优先级与全部环境变量
- [Deployment-and-Operations.md](Deployment-and-Operations.md)：反代、备份、升级
- [Troubleshooting.md](Troubleshooting.md)：常见故障
- [docs/ops-deploy-checklist.md](../docs/ops-deploy-checklist.md)：生产部署前后检查清单
