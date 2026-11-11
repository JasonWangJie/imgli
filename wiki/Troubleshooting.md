# 故障排查

[← 返回 Wiki 首页](Home.md)

## 通用排查顺序

```text
1. imgli doctor                          # 有 FAIL → exit 1；仅 WARN → exit 0
2. curl -fsS http://localhost:8686/healthz
3. imgli version / 管理后台「系统」        # 确认实际运行版本与构建形态
4. 核对 IMGLI_BASE_URL、IMGLI_TRUST_PROXY、IMGLI_DATABASE_*（见 Configuration.md）
5. 看 log/slog 结构化日志与 GORM 慢查询告警
6. 对照 CHANGELOG.md 中对应版本的修复记录，判断是否属于已修复的旧版本问题
```

## 高频问题

### 反代后登录 / 注册 403「跨站请求被拒绝」

用 1Panel / Nginx / Caddy 反代到域名后，登录注册 403，而 `http://IP:端口` 正常。
根因通常是 `IMGLI_BASE_URL` 仍是 localhost 或 IP:端口，与浏览器实际访问的
`https://域名` 不一致；TLS 在反代终止时，应用侧无法仅靠 `Host` 判定同源。

```bash
IMGLI_BASE_URL=https://img.example.com
IMGLI_TRUST_PROXY=true     # 仅在可信反代之后打开
```

详见 [Configuration.md](Configuration.md) 与
[docs/security-hardening.md](../docs/security-hardening.md#faq-reverse-proxy-loginregister-cross-site-rejected)。
v0.7+ 可在管理后台「系统 / 运维」看同源健康报告。

### 前台白屏 / 显示「前端未构建」提示页

`web/dist` 不存在或构建产物未编入二进制。执行 `make build`（= web + build-go）
或 `npm run build` 后重新构建；容器部署请直接使用官方镜像。
部署前后检查见 [docs/ops-deploy-checklist.md](../docs/ops-deploy-checklist.md)。

### HEIC/HEIF 上传返回 `415 heic_unsupported`

纯 Go 归档（GitHub Release、`make build`）不带 HEIC 解码能力；官方 Docker 镜像与
`make build-vips` 带 libvips + libheif，会把 HEIC 解码为 JPEG 后走现有管线。
另外检查用户组 `allowed_exts` 是否包含 `heic/heif`——schema v9 只为
「恰好是旧默认五后缀」的存量组自动追加，`imgli doctor` 会给出 WARN。
见 [Overview.md](Overview.md) 的「两种构建形态」。

### 图库缩略图 NOT FOUND / 第一次加载 JSON 500

MinIO/S3 上传后首次 `/t` 可能因 HEAD 403/503 或缺少 Content-Length 失败；
后台格子对走 CDN 的公开图也可能找不到源站缩略图。v0.9.15 / v0.9.16 已修复
（`Open` 回退 GET、缺缩略图从原图补、`/t` 回源 CDN 对象、前端 `<img>` 用
`?r=1` 重试一次）。若仍出现，先确认运行版本 ≥ v0.9.16，再检查 CDN 回源策略。

### `429 rate_limited`

限速有两类：接口级固定窗口（config 60/min、plaza/share 120/min、unlock 30/min、
auth 20/min 等）与上传的用户组分/时/日三档限速。客户端应按 `Retry-After` 退避；
上传队列会自动重试 `rate_limited`。`IMGLI_RATE_LIMIT_MULT` 只用于测试环境放大，
生产保持 `1`；要长期调整请改管理后台的用户组配置。

### URL 抓取上传被拒绝

`/upload/url` 默认拒绝内网目标，并做域名解析与实际连接的双重 SSRF 校验。
确需放行时用 `IMGLI_FETCH_ALLOW` 精确添加 host / CIDR，不要整体放开。
见 [Authentication-and-Security.md](Authentication-and-Security.md)。

### SMTP 530/535、测试发信与表单不符

v0.9.14 起支持 AUTH LOGIN（飞书/Lark、部分 163/QQ 端点只声明 LOGIN），
测试发信使用当前表单值而非上次保存的配置，并把 530/535、TLS/端口不匹配、
超时等错误映射为可操作提示。修改用户名后密码掩码会被清空，需重新输入。
仍失败时先用管理后台「Email SMTP」的预览/测发定位。

### SQLite 绑定挂载 OOM / 权限错误

v0.9.6 起针对低内存与绑定挂载场景做了缓解，Docker entrypoint 会修正
`imgli.db` / `-wal` / `-shm` 的属主再降权。不要把 SQLite 文件放在 NFS / CIFS 上；
生产可考虑切 PostgreSQL（`IMGLI_DATABASE_DRIVER=postgres`）。
备份与恢复见 [docs/backup.md](../docs/backup.md)。

### OIDC 登录失败 / 回调异常

检查 issuer / client id / redirect URI 与 `IMGLI_BASE_URL` 是否一致，
按 [docs/oidc-operator.md](../docs/oidc-operator.md) 的 checklist 与 failure modes 排查。

### 私密相册出现在广场 / 匿名分享

v0.9.12 起私密相册不再漏到广场、匿名 `/i` 与 `/s`。若仍可见，先确认版本，
再检查相册可见性与图片 surface 字段。

### 访问统计与 CDN 报表对不上

访问统计只统计经过源站 `/i` 门禁的请求，CDN 边缘命中不会计入。
见 [deploy/ops/README.md](../deploy/ops/README.md) 与
[docs/ops-cleanup-cdn-boundary.md](../docs/ops-cleanup-cdn-boundary.md)。

### 升级 / 清理 / 搬迁后数据异常

先查 [docs/user-groups-lifecycle.md](../docs/user-groups-lifecycle.md)（组级保留与
强制存活）、[docs/ops-cleanup-cdn-boundary.md](../docs/ops-cleanup-cdn-boundary.md)
（清理边界）、[docs/storage-migrate.md](../docs/storage-migrate.md)（搬迁续跑）。
清理务必先跑 `/admin/cleanup/preview` 再 `run`；搬迁任务可 cancel / resume，
已复制对象不会回滚。

相关：[Deployment-and-Operations.md](Deployment-and-Operations.md) ·
[Configuration.md](Configuration.md) · [CHANGELOG.md](../CHANGELOG.md)
