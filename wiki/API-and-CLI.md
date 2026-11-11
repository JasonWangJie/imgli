# HTTP API 与 CLI

[← 返回 Wiki 首页](Home.md) · 端点权威定义在
[internal/server/server.go](../internal/server/server.go) 的 `mountAPI()` / `mountServe()`；
逐条端点清单（107 个 `/api/v1` 端点、错误码全表）见 [接口文档.md](../接口文档.md)。
本页只做分组概览，不复制那张完整清单。

## 通用约定

| 项 | 约定 |
|---|---|
| 业务 API 前缀 | `/api/v1` |
| 响应信封 | 成功 `{status:true, message:"ok", data:<任意>}`；失败 `{status:false, message:<中文提示>, data:{"code":"..."}}`（`internal/handler/respond.go`）。**机器判断读 `data.code`**，不要解析 `message` |
| 错误码 | `respond.go` 常量：`file_too_large`、`quota_exceeded`、`ext_not_allowed`、`rate_limited`、`moderation_rejected`、`unauthorized`、`forbidden`、`not_found`、`invalid_request`、`internal_error`、`resource_gone`、`bandwidth_exceeded`、`expires_over_group`、`max_views_over_group`、`heic_unsupported` |
| 信封例外 | `GET /api/v1/admin/export/users.csv` 返回 `text/csv` 文件流 |
| 认证 | `Authorization: Bearer <API_TOKEN>` 优先，其次 Cookie `imgli_session` |
| CSRF | 写方法走 `OriginCheck`；Bearer 请求放行 |
| 限速 | 进程内固定窗口，响应用 `Retry-After`；上传按用户组分/时/日三档 |
| 图片直链 | 不在 `/api/v1` 下：`/i/{name}`、`/t/{name}?w=...`、`/avatar/{id}` |
| 存活探针 | `GET /healthz` |

## 端点分组

| 组 | 典型端点 | 门禁 |
|---|---|---|
| 公开配置 / 发现 | `GET /config`、`/plaza`、`/plaza/albums`、`/u/{username}`、`/u/{username}/images`、`/u/{username}/albums`、`/s/{key}`、`POST /s/{key}/unlock`、`/a/{id}`、`/a/{id}/images`、`POST /a/{id}/unlock` | 匿名 + IP 限速 |
| 账号 | `POST /auth/register`、`/auth/login`、`/auth/logout`、`/auth/forgot-password`、`/auth/reset-password`、`/auth/verify-email`、`/auth/confirm-change-email`、`GET /auth/oidc/start`、`/auth/oidc/callback` | 匿名 + 命名桶限速 |
| 用户私有 | `GET /auth/session`、`POST /auth/resend-verification`、`/user/profile`(GET/PATCH)、`/user/email/change`、`/user/preferences`、`/user/avatar`、`/user/watermark`、`/user/password`、`DELETE /user`、`/user/quota`、`/user/policies`、`/user/tokens`(GET/POST/DELETE/{id}) | `RequireUser` + `RequireFullScope` |
| 图片 | `GET /images`、`POST /images/batch`、`GET/PATCH/DELETE /images/{key}`、`GET /images/{key}/stats` | 登录用户 |
| 回收站 | `GET /trash`、`DELETE /trash`、`POST /trash/{key}/restore`、`DELETE /trash/{key}/purge` | 登录用户 |
| 相册 | `/albums` CRUD、`/{id}/stats`、`/{id}/reorder`、`/{id}/images/visibility` | 登录用户 |
| 上传 | `POST /upload`、`POST /upload/url` | `RequireUserOrAnon` + 用户组限速 |
| 管理后台 | `/admin/stats`、`/admin/referers/images`、`/admin/users`、`/admin/export/users.csv`、`/admin/users/{id}`、`/admin/images*`、`/admin/review*`、`/admin/groups*`（含 lifecycle preview/apply）、`/admin/policies*`（含 `/{id}/test`）、`/admin/storage/migrate*`、`/admin/system/{version,health,check-update,upgrade}`、`/admin/cleanup/{preview,run}`、`/admin/settings`（含 smtp/test、mail/preview、mail/test、moderation/test）、`/admin/logs`、`/admin/invites*`、`/admin/webhooks`、`/admin/oidc` | `RequireAdmin` |

## 上传 API

```bash
curl -X POST https://your-host/api/v1/upload \
  -H "Authorization: Bearer <API_TOKEN>" \
  -F file=@shot.png \
  -F visibility=public
```

表单字段（前端 `web/src/upload/uploader.ts`）：
`file`、`visibility`、`album_id`、`policy_id`、`expires_in`、`max_views`。

响应 `data.links` 含 `url`、`markdown`、`html`、`bbcode`、`thumbnail_url`，
由 `internal/linkbuilder` 按 `IMGLI_BASE_URL` 拼接。

`POST /upload/url` 抓取远端 URL，服务端会做 SSRF 双重校验；该通道没有上传进度。
浏览器上传是**单请求 multipart 直传，不是分片**；使用 XHR 仅为获取进度事件。

Token 在「设置 → API Token」创建，明文只显示一次，scope 为 `upload` 或 `full`：
`upload` scope 只能调上传接口，其它私有接口会被 `RequireFullScope` 拒绝。

## CLI 子命令

```text
imgli <serve|upload|import-dir|doctor|migrate|storage-migrate|version> [flags]
```

| 命令 | 用途 |
|---|---|
| `serve` | 启动服务（加载配置 → 打开 DB → 迁移 → Seed → 启动 HTTP 与后台任务） |
| `upload` | 上传本地文件或 stdin |
| `import-dir` | 把本地目录批量导入到上传 API |
| `doctor` | 自检 data 目录 / DB / base_url / trust_proxy / 监听 / local 策略写读删 |
| `migrate` | 手动执行数据库迁移与 Seed |
| `storage-migrate` | 跨策略对象搬迁，支持 `-from -to -dry-run -delete-source -limit -config` |
| `version` | 打印 ldflags 注入的版本 |

`upload` / `import-dir` 读取 `IMGLI_BASE_URL` 与 `IMGLI_TOKEN`：

```bash
export IMGLI_BASE_URL=https://your-host
export IMGLI_TOKEN='粘贴一次性显示的 token'
imgli upload shot.png                              # → https://your-host/i/….png
imgli upload -verbose shot.png                    # stderr 打印组有效期/次数限制
imgli upload -format markdown shot.png            # url | markdown | json
imgli upload -format json -visibility private shot.png
imgli upload -expires-in 86400 shot.png
cat shot.png | imgli upload -name shot.png -      # stdin
```

其它参数：`-base-url`、`-token`、`-name`（stdin 时指定文件名）。
`-verbose` 输出的用户组策略含义见
[docs/user-groups-lifecycle.md](../docs/user-groups-lifecycle.md)。

## 客户端集成

| 客户端 | 文档 |
|---|---|
| PicGo / Typora / VS Code | [docs/picgo.md](../docs/picgo.md) |
| ShareX | [docs/integrations/sharex.md](../docs/integrations/sharex.md) |
| uPic | [docs/integrations/upic.md](../docs/integrations/upic.md) |
| 索引 | [docs/integrations/README.md](../docs/integrations/README.md) |

出站 Webhook 事件带 HMAC 签名，管理端 `GET/PUT /admin/webhooks` 配置；
OIDC 运维排障见 [docs/oidc-operator.md](../docs/oidc-operator.md)。

相关：[Authentication-and-Security.md](Authentication-and-Security.md) ·
[Storage-and-Media.md](Storage-and-Media.md) · [Frontend.md](Frontend.md)
