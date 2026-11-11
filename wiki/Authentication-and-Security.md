# 认证与安全

[← 返回 Wiki 首页](Home.md) · 加固清单见 [docs/security-hardening.md](../docs/security-hardening.md)

## 认证机制

**imgli 不使用 JWT。** 会话与 Token 都是不透明随机串，数据库只存 SHA-256 哈希
（`internal/token/token.go`）。

| 机制 | 细节 |
|---|---|
| 浏览器会话 | Cookie `imgli_session`，`HttpOnly` + `SameSite=Lax`；`BaseURL` 为 https 时加 `Secure`；TTL 30 天。`Session` 表主键即 token 哈希 |
| API Token | 明文 `bl_` + 32 字节随机串，scope `upload` \| `full`；**仅创建时返回一次**。`/user/tokens` 管理；`upload` scope 仅能上传，`RequireFullScope` 会拦截其它私有接口 |
| OIDC SSO | 可选。Authorization Code + `id_token` 校验，按 email find-or-create；配置存 settings 键 `oidc`。端点 `/api/v1/auth/oidc/start`、`/callback`，管理端 `GET/PUT /admin/oidc` |
| 密码 | argon2id（m=64MB, t=1, p=4，PHC 格式，常数时间比较） |
| 邮箱流程 | 验证 / 重置 / 改绑使用一次性 `AuthToken`（TTL 1h / 24h） |
| 分享口令 | 图片访问口令用 argon2id 哈希，解锁成功后写 HMAC Cookie / Header |
| 注册 | 模式 `open` \| `invite` \| `closed`；游客上传由 `guest_upload_enabled` 控制 |

前端**不持久化会话 token**：`fetch` 同源自动携带 Cookie，前端不设置
`Authorization`；`localStorage` 只存主题、语言、视图等偏好
（见 [Frontend.md](Frontend.md)）。

## 权限矩阵（服务端为安全边界）

| 路由组 | 门禁 | 说明 |
|---|---|---|
| `/api/v1/config`、`/plaza*`、`/u/{username}`、`/s/{key}`、`/a/{id}` | 匿名 + 按 IP 固定窗口限速 | 配置 60/min；广场/分享 120/min；解锁 30/min |
| `/auth/register`、`/login`、`/reset-password` 等 | 匿名 + 命名桶限速 | auth 20/min、resend 3/min、forgot 5/min |
| `/auth/session`、`/user/*`、`/images/*`、`/trash/*`、`/albums/*` | `RequireUser` + `RequireFullScope` | Session 或 full-scope Token；upload-only Token 403 |
| `/upload`、`/upload/url` | `RequireUserOrAnon` + `GroupMiddleware` | 登录（Session / 任意 scope Bearer）或真匿名；**出示无效凭证返回 401**，防止被静默当成游客；限速按用户组分/时/日三档 |
| `/admin/*` | `RequireAdmin` | `User.IsAdmin`；首个注册用户即管理员 |
| `/i/{name}`、`/t/{name}`、`/avatar/{id}` | `Auth` + `servesvc` 门禁 | 匿名也过 Auth（解析身份），再由直链门禁判定可见性；管理员可预览他人私密/待审图 |

前端的 `RequireAuth` / `RequireAdmin` 只是体验层，安全边界全部在后端
（`internal/handler/authmw.go`、`adminmw.go`）。

## 中间件链

| 中间件 | 作用 |
|---|---|
| `Recoverer` | panic → `slog.Error` + 500 |
| `RealIP` | 仅 `trust_proxy` 为真时取 `X-Forwarded-For` 最右值 |
| `OriginCheck` | 写方法同源校验；Bearer 请求放行（配合 `SameSite=Lax` 防 CSRF） |
| `Auth` | Bearer 优先，其次 `imgli_session` Cookie |
| `RequireUser` / `RequireUserOrAnon` / `RequireFullScope` / `RequireAdmin` | 身份与 scope 门禁 |
| `Limiter` | 进程内固定窗口；`Middleware` 按用户/IP、`IPMiddleware` 恒按 IP、`GroupMiddleware` 读 DB 用户组三档限速 |
| `ValidateFetchURL` + `DialContext` | URL 抓取的 SSRF 双重校验（域名解析与实际连接都拦） |
| `imgpass` | 图片口令 HMAC Cookie |
| `singleflight` | 合并并发缩略图生成，防缓存击穿 |

## 常见安全边界

- **CSRF**：`OriginCheck` 依赖正确的 `IMGLI_BASE_URL`；反代 TLS 终止时必须把
  Base URL 设为浏览器实际访问的公网源，并开 `IMGLI_TRUST_PROXY`。
- **SSRF**：`/upload/url` 默认拒绝内网地址，需要时用 `IMGLI_FETCH_ALLOW` 按
  host / CIDR 精确放行，不要整体放开。
- **私图不可枚举**：私密相册不会出现在广场 / 匿名 `/i` / `/s`；private 对象键
  的 CDN 302 fail-closed，只走 60s 预签名。
- **CDN 与统计**：访问统计只覆盖源站 `/i` 门禁，CDN 边缘命中不可见，
  见 [deploy/ops/README.md](../deploy/ops/README.md) 与
  [docs/ops-cleanup-cdn-boundary.md](../docs/ops-cleanup-cdn-boundary.md)。
- **审计**：管理操作写 `AuditLog`，可在 `GET /api/v1/admin/logs` 查询。
- **Secrets**：PR 中不得出现真实凭据、生产主机名；安全报告走
  [SECURITY.md](../SECURITY.md)，不要开公开 issue。

## 排障入口

| 现象 | 先看 |
|---|---|
| 反代后登录/注册 403「跨站请求被拒绝」 | [Troubleshooting.md](Troubleshooting.md) · [docs/security-hardening.md](../docs/security-hardening.md#faq-reverse-proxy-loginregister-cross-site-rejected) |
| OIDC 登录失败 / 回调异常 | [docs/oidc-operator.md](../docs/oidc-operator.md) |
| 限速误伤 | `IMGLI_RATE_LIMIT_MULT`（仅测试环境放大）、用户组三档限速配置 |
