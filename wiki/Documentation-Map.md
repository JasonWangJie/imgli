# 文档地图与同步规范

[← 返回 Wiki 首页](Home.md) · 分层规则 SSOT：
[docs/documentation-ssot.md](../docs/documentation-ssot.md) ·
协作规则：[AGENTS.md](../AGENTS.md)

## 三层文档

| 层 | 位置 | 受众 | 放什么 |
|---|---|---|---|
| 工程（Git） | 本仓库 `docs/`、`README*`、[`wiki/`](Home.md)、`CHANGELOG.md` | 贡献者、读源码的自托管者 | 与代码 / tag 绑定的矩阵、脚本、设计稿、运维清单、架构说明 |
| 产品 | 维护者 KB `docs-imgli/*` → [docs.imgli.com](https://docs.imgli.com) | 最终用户 / 产品 UI 操作者 | how-to、FAQ、简短特性说明；长矩阵链接回 GitHub |
| 内部 | 维护者 KB `imgli/*`、`hub/imgli` | 维护者 | Roadmap、生产运维细节、商业策略；**不进公开仓库** |

**一次职责只在一处编辑**：摘要与链接可以有多份，完整副本不要有多份。
冲突时以 git tag + `CHANGELOG.md` + 最新 Release 为准。

## 仓库根目录

| 文件 | 用途 |
|---|---|
| [README.md](../README.md) / [README.zh-CN.md](../README.zh-CN.md) | 项目主页：特性、对比、快速开始、配置、上传 API、文档索引 |
| [readmenew.md](../readmenew.md) | 代码变更说明入口 + 文档导航（按 `AGENTS.md` 每次变更同步） |
| [接口文档.md](../接口文档.md) | 手写 API 参考：107 个 `/api/v1` 端点、错误码全表、上传契约、非 API HTTP 表面（仓库没有 OpenAPI/Swagger 定义） |
| [AGENTS.md](../AGENTS.md) | 项目协作规则：变更必须同步的三项 |
| [CONTRIBUTING.md](../CONTRIBUTING.md) | 贡献许可、开发环境、文档规则、版本与发版 |
| [CHANGELOG.md](../CHANGELOG.md) | 各版本变更记录；`[Unreleased]` 放用户可见变更 |
| [ROADMAP.md](../ROADMAP.md) | 公开路线图镜像（执行面为 GitHub Issues） |
| [SECURITY.md](../SECURITY.md) | 安全报告渠道 |
| [COMMERCIAL.md](../COMMERCIAL.md) · [LICENSE](../LICENSE) · [NOTICE](../NOTICE) | 商业双许可、AGPL-3.0-only、第三方说明 |
| [Makefile](../Makefile) · [Dockerfile](../Dockerfile) · [docker-compose.yml](../docker-compose.yml) · [docker-entrypoint.sh](../docker-entrypoint.sh) | 构建与部署 |
| [go.mod](../go.mod) · [go.sum](../go.sum) | Go 依赖 |
| [.github/pull_request_template.md](../.github/pull_request_template.md) | PR 检查清单（三项文档同步、测试、DCO、无 secrets） |

## `docs/` 工程文档

| 文件 | 主题 |
|---|---|
| [documentation-ssot.md](../docs/documentation-ssot.md) | 文档分层与「一处编辑」规则 |
| [s3-compatibility.md](../docs/s3-compatibility.md) | S3 兼容矩阵与策略字段（CDN / 前缀 / 路径风格 / 路径模板） |
| [webdav-compatibility.md](../docs/webdav-compatibility.md) · [storage-ftp.md](../docs/storage-ftp.md) | WebDAV 矩阵、FTP 双轨与限制 |
| [storage-migrate.md](../docs/storage-migrate.md) | 跨策略存储搬迁（CLI + 管理端任务） |
| [user-groups-lifecycle.md](../docs/user-groups-lifecycle.md) | 用户组有效期 / 次数上限 / 保留与强制存活 |
| [ops-deploy-checklist.md](../docs/ops-deploy-checklist.md) | 生产部署前后检查、契约护栏、回滚、版本升级说明 |
| [ops-release.md](../docs/ops-release.md) | 发版 / CI / 部署 playbook |
| [ops-cleanup-cdn-boundary.md](../docs/ops-cleanup-cdn-boundary.md) | 清理 kinds、CDN 与配额边界 |
| [security-hardening.md](../docs/security-hardening.md) | 加固清单与反代 / CSRF FAQ |
| [backup.md](../docs/backup.md) | SQLite / PostgreSQL 备份与恢复 |
| [picgo.md](../docs/picgo.md) · [integrations/](../docs/integrations/README.md) | PicGo / ShareX / uPic 客户端配置 |
| [oidc-operator.md](../docs/oidc-operator.md) · [moderation-spot-check.md](../docs/moderation-spot-check.md) | OIDC 排障、机审抽检 |
| [design/](../docs/design/) | 设计稿：存储能力分级、站点定制 IA、搬迁同步等 |
| [superpowers/](../docs/superpowers/) | 计划与验收记录（如 v0.9.2 验收、HEIC 计划） |
| [lexicon/](../docs/lexicon/) · [brand/](../docs/brand/README.md) · [community/](../docs/community/README.md) · [screenshots/](../docs/screenshots/README.md) | 术语、品牌素材、社区文案、截图 |
| [license-relicensing-checklist.md](../docs/license-relicensing-checklist.md) | 许可变更检查清单 |

## `deploy/` 与 `scripts/`

| 位置 | 主题 |
|---|---|
| [deploy/imgli.example.yaml](../deploy/imgli.example.yaml) | 运维配置样例 |
| [deploy/compose.prod.example.yml](../deploy/compose.prod.example.yml) · [Caddyfile.example](../deploy/Caddyfile.example) · [traefik.labels.example.yml](../deploy/traefik.labels.example.yml) · [imgli.service.example](../deploy/imgli.service.example) | 生产 Compose、TLS 反代、systemd |
| [deploy/ocr-paddle/](../deploy/ocr-paddle/README.md) | 外置 OCR 旁路服务 |
| [deploy/ops/](../deploy/ops/README.md) | 备份、健康检查、CDN/Nginx 片段与计量说明 |
| [scripts/](../scripts/) | 安装、冒烟、压测、厂商矩阵、字体子集、发版门禁；索引见 [Deployment-and-Operations.md](Deployment-and-Operations.md) |

## 本 Wiki 页面

[Home.md](Home.md) 是索引。主题页：
[Overview](Overview.md) · [Architecture](Architecture.md) · [Getting-Started](Getting-Started.md) ·
[Configuration](Configuration.md) · [Authentication-and-Security](Authentication-and-Security.md) ·
[Storage-and-Media](Storage-and-Media.md) · [API-and-CLI](API-and-CLI.md) ·
[Frontend](Frontend.md) · [Development-and-Testing](Development-and-Testing.md) ·
[Deployment-and-Operations](Deployment-and-Operations.md) · [Troubleshooting](Troubleshooting.md)。

Wiki 定位为**索引 + 结构化说明**，不复刻 `docs/` 里的大矩阵，也不复制 README 的
完整特性列表或 [接口文档.md](../接口文档.md) 的逐条端点表；主题细节一律链接到 SSOT。
接口变更时，`接口文档.md` 的「端点总览」表与
[API-and-CLI.md](API-and-CLI.md) 的分组概览需同步更新。

## 每次代码变更要同步什么

依据 [AGENTS.md](../AGENTS.md) 与
[.github/pull_request_template.md](../.github/pull_request_template.md)：

| # | 目标 | 内容 |
|---|---|---|
| 1 | [`readmenew.md`](../readmenew.md) | 本次变更对应的说明 |
| 2 | 本 Wiki 中受影响的页面 | 最新行为、使用方式或维护事项（PR 描述里列出页面链接） |
| 3 | 项目变更台账 | 日期、变更内容、影响范围、验证结果（PR 描述里列出条目链接） |

三项都是必需项，包含修复、重构、内部调整；仓库内文档更新必须进同一个 PR。
此外还要按 [CONTRIBUTING.md](../CONTRIBUTING.md) 更新受影响的 `docs/`、
`README*` 与 `CHANGELOG.md` 的 `[Unreleased]`。

### 入口定位状态

- `readmenew.md` 与本 Wiki 原先在仓库内不存在，本次任务已建立为仓库内入口
  （见 [readmenew.md](../readmenew.md) 的变更记录）。
- **项目变更台账**：在本仓库内未定位到对应文件；若维护在维护者 KB 或其它外部系统，
  需要维护者提供入口与访问权限。按 `AGENTS.md` 第 17 行，无法定位或没有权限时
  必须在交付说明中明确报告该项未完成，**不得声称已同步，也不得用其它文档替代**。
- 本次不新建替代性台账文件。
