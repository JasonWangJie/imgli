# 存储与媒体处理

[← 返回 Wiki 首页](Home.md)

本页是概要与导航。**厂商兼容矩阵、策略字段、搬迁细节的 SSOT 在
[docs/](../docs/)**，不要在 Wiki 或产品文档里维护第二份完整矩阵
（见 [docs/documentation-ssot.md](../docs/documentation-ssot.md)）。

## 存储驱动

统一接口 `Put / Open / Delete / Exists`（`internal/storage/storage.go`），
可选 `Presigner` 生成 60s 私有对象预签名 URL。`storagesvc.Resolver` 按
`policyID + 配置指纹` 缓存驱动实例，配置变更即失效。

| 驱动 | 实现 | 能力分级 | 适用 |
|---|---|---|---|
| `local` | 原子临时文件写、拒绝越界键 | `first_class` | 单默认策略，零外部依赖 |
| `s3` | **手写 AWS SigV4**（无 SDK）、`Authorization` 头、预签名 | `first_class` | MinIO / RustFS / R2 / OSS / COS 等 S3 兼容端点；CDN 302 |
| `webdav` | 标准库 `net/http` + Basic，禁 endpoint 内联 userinfo | `supported` | OpenList / 网盘出口 |
| `ftp` | FTP / FTPS 显式与隐式 TLS、连接池 | `compat` | 兼容层：无 Range / CDN / 预签名，不作热路径 |

> `model.StoragePolicy.Driver` 的注释里仍出现 `oss|cos`，但
> `storagesvc.Resolver` 实际只识别 `local/s3/webdav/ftp`；OSS/COS 请按
> S3 兼容 endpoint 配置。

对象键按 `public/` 与 `private/` surface 前缀隔离。CDN 302 目标由
`ObjectURL` 生成，private 键 fail-closed 不生成 CDN 链接；私密图走预签名直连。

| 主题 | SSOT |
|---|---|
| S3 厂商矩阵与策略字段（CDN / 前缀 / 路径风格 / 路径模板） | [docs/s3-compatibility.md](../docs/s3-compatibility.md) |
| WebDAV 厂商矩阵 | [docs/webdav-compatibility.md](../docs/webdav-compatibility.md) |
| FTP 双轨与限制 | [docs/storage-ftp.md](../docs/storage-ftp.md) |
| 驱动能力分级设计 | [docs/design/storage-caps-draft.md](../docs/design/storage-caps-draft.md) |
| 跨策略搬迁 | [docs/storage-migrate.md](../docs/storage-migrate.md) |

## 跨策略搬迁

- 管理后台：`POST /api/v1/admin/storage/migrate` 启动，另有 list / get /
  cancel / resume 端点；任务落库为 `StorageMigrateJob`，重启从
  `cursor_after_id` 续跑，size 校验，可取消（已复制对象不回滚）。
- CLI：`imgli storage-migrate -from <id> -to <id> [-dry-run] [-delete-source] [-limit N]`，
  前台同步执行。
- 前端对未完成任务以 1s 轮询显示进度（这是全站唯一的周期轮询，
  没有 SSE / WebSocket）。

## 缩略图与直链

| 项 | 行为 |
|---|---|
| 原图 | `GET /i/{name}` |
| 缩略图 | `GET /t/{name}?w=120\|200\|240\|400\|480\|800\|960\|1600`（宽度白名单） |
| 头像 | `GET /avatar/{id}`（本地 `{data_dir}/avatars`） |
| 键布局 | surface 前缀 + `.thumbs/g1/{hash}.{jpg\|webp}`，读取时 dual-probe 兼容旧布局 |
| 缺失补生成 | 默认缩略图缺失时从原图补；S3 上传后首次 `/t` 可回退 GET 拉取原图 |
| 并发保护 | `singleflight` 合并同一缩略图的并发生成 |
| 错误响应 | 图片请求 5xx 返回 SVG 占位图 + `Cache-Control: no-store`，避免反代把 JSON 错误缓存成 JPEG |

`/i` 可 302 到 CDN，因此管理后台缩略图走 `/t`；v0.9.16 起 `/t` 在源站未命中时
会拉取 `/i` 会重定向到的同一 CDN 对象再生成缩略图。

## 上传处理管线

`internal/imaging/pipeline.go` 的 `ProcessPipeline` 采用**一次解码**：

```text
解码（安全上限 24MiB / 16MP）
  → max_edge 等比缩
  → 文字水印 / PNG 图像水印（内嵌 Noto Sans SC 子集，校验字形覆盖）
  → 末次编码（JPEG 质量默认 90；可选原图转 WebP，仅 vips 构建）
```

- `strip_exif` 默认开启；`webp_skip_if_larger` 在 WebP 反而更大时回退原编码。
- JPEG 质量 / 原图 WebP 由管理后台「设置 → 图片处理」配置。
- 纯 Go 缩略图为 JPEG Q82（CatmullRom 缩放、白底平铺 alpha）；
  vips 构建输出 WebP 缩略图。
- 同用户秒传：`File` 按 `(hash, surface)` 唯一，命中后复用，不二次扣配额。

## HEIC / HEIF

魔数嗅探 `SniffHEIF` 识别上传；解码需要 **libvips + libheif**（-tags vips），
解码后存为 JPEG 再走现有管线。纯 Go 归档返回 `415 heic_unsupported`。
v0.9.18 的 schema v9 会为「恰好是旧默认五后缀」的存量用户组一次性追加
`heic/heif`，自定义后缀列表不受影响。

## 内容安全（机审）

| 组件 | 说明 |
|---|---|
| NSFW scorer | 可插拔：webhook / nsfwjs / openai / aliyun / tencent；结果进入审核队列 `pending \| rejected`，触发拒审邮件与审计 |
| OCR 词表 | 项目**不内置 OCR 引擎**。`ocr_keywords` 插件把原图 POST 给外置 HTTP OCR 服务，返回 `{"text": ...}` 后做大小写不敏感词表匹配 |
| 旁路服务 | [deploy/ocr-paddle/](../deploy/ocr-paddle/README.md) 提供 RapidOCR / PaddleOCR 三种 Python server + Dockerfile，契约 `POST` 原图 → `{"text"}`，`GET /health` |
| 抽检 | [docs/moderation-spot-check.md](../docs/moderation-spot-check.md) |
| Webhook | 出站事件 HMAC 签名，异步投递 |

## 生命周期与清理

- `Image` 软删即回收站；可恢复或彻底删除（purge）。
- 图片可设置过期时间、最大查看次数、访问口令（阅后即焚）。
- 用户组支持组级有效期/次数上限、保留与强制存活；每小时执行清理，
  管理后台提供 `/admin/cleanup/preview` 与 `/admin/cleanup/run`（dry-run + 确认）。
- 边界说明（清理不做什么、CDN 与配额的关系）：
  [docs/ops-cleanup-cdn-boundary.md](../docs/ops-cleanup-cdn-boundary.md)；
  组策略细节：[docs/user-groups-lifecycle.md](../docs/user-groups-lifecycle.md)。

相关：[API-and-CLI.md](API-and-CLI.md) ·
[Deployment-and-Operations.md](Deployment-and-Operations.md) ·
[Troubleshooting.md](Troubleshooting.md)
