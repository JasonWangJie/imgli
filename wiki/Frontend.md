# 前端（React SPA）

[← 返回 Wiki 首页](Home.md) · 代码在 [web/](../web/)

## 技术栈（[web/package.json](../web/package.json)）

| 类别 | 选型 |
|---|---|
| 框架 | React 19.1、React Router 7（`BrowserRouter` 在 `web/src/main.tsx`） |
| 构建 | Vite 7 + `@vitejs/plugin-react`、TypeScript 5.8（strict / noEmit / ES2022） |
| 样式 | Tailwind CSS v4 + `@tailwindcss/vite`，设计令牌在 `src/styles/tokens.css`；**无第三方 UI 组件库** |
| 服务端状态 | `@tanstack/react-query` 5（query key 集中在 `src/api/queryKeys.ts`） |
| 客户端状态 | `zustand` 5（`src/store.ts`、`src/upload/queue.ts`） |
| 图表 | `chart.js` + `react-chartjs-2`（管理后台趋势图） |
| 字体 | `@fontsource/manrope` |
| 测试 | Vitest 3 + Testing Library + jsdom；Playwright e2e（13 个 spec） |
| PWA | `public/manifest.webmanifest`、`public/sw.js`，仅生产注册 |

常用命令：

```bash
cd web
npm run dev          # Vite 开发服务器
npm run build        # tsc --noEmit && vite build → web/dist
npm test             # vitest run
npm run e2e:smoke    # 4 个核心 spec
npm run e2e          # 全量 Playwright（会先构建二进制）
```

## 目录结构

| 路径 | 职责 |
|---|---|
| `src/main.tsx` | StrictMode + QueryClientProvider + BrowserRouter + 主题 + SW 注册 |
| `src/App.tsx` | **全部前台路由**、全局 401/403 回调、站点标题/favicon/主题注入 |
| `src/api/` | `client.ts`（fetch 封装）、`hooks.ts`、`adminHooks.ts`、`queryKeys.ts`、`types.ts`（约 69 个类型） |
| `src/pages/` | `about/ admin/ albums/ auth/ discover/ images/ settings/ share/ trash/ upload/` |
| `src/pages/admin/` | 懒加载后台：`AdminApp.tsx`、`shell/`、`dashboard/ users/ images/ review/ groups/ invites/ policies/ system/ settings/ logs/` |
| `src/shell/` | `AppLayout`、`GuestLayout`、`Nav`、`RequireAuth` / `RequireAdmin` / `RequireAuthOrGuest` 等壳 |
| `src/ui/` | 自研基础组件（Button / Input / Modal / Segmented / Toggle / Toasts / RetryImg / SiteSlots…） |
| `src/upload/` | `uploader.ts`（XHR 直传）、`queue.ts`（zustand 队列） |
| `src/lib/` | 纯工具与页面级 hooks：`cn.ts`、`format.ts`、`siteTheme.ts`、`safeNext.ts` 等（**没有独立 `hooks/` 目录**） |
| `src/i18n/` | `t()` 点路径 + `{var}` 插值；zh/en 各 20 个 locale 模块 |
| `src/styles/` | Tailwind v4 `@theme inline` + `:root` / `body[data-theme='dark']` CSS 变量 |
| `public/` | PWA manifest / sw.js / brand 图标（明暗两套） |
| `e2e/` | 13 个 Playwright spec |

注意：`clsx` 与 `tailwind-merge` 虽列在 devDependencies，但被
`src/lib/cn.ts` 在运行时 import。

## 路由与权限

路由集中在 [`web/src/App.tsx`](../web/src/App.tsx)（约 110–156 行），
后台子路由在 `src/pages/admin/AdminApp.tsx`。

| 类别 | 路径 |
|---|---|
| 公开 | `/login`、`/about`、`/forgot-password`、`/reset-password`、`/verify-email`、`/confirm-email`、`/explore`、`/u/:username`、`/a/:id`、`/s/:key` |
| 游客或登录 | `/`（上传页）：登录用 `AppLayout`，游客用 `GuestLayout` + GUEST 徽标；是否真能上传由后端 `guest_upload_enabled` 与游客组限额决定 |
| 必须登录 | `/images`、`/albums`、`/albums/:id`、`/trash`、`/settings/:tab?`（`profile / preferences / tokens / usage`） |
| 管理员 | `/admin`（仪表盘）、`users`、`images`、`review`、`groups`、`invites`、`policies`、`system`、`settings`、`logs` |

- 全局 401 → `/login?next=当前路径`（排除 `/auth/session` 探测与登录失败）；
  403 在后台提示「权限被收回」并回首页。登录回跳走 `src/lib/safeNext.ts` 白名单，
  防 open redirect。
- 前端权限壳只是体验层，安全边界在后端（见
  [Authentication-and-Security.md](Authentication-and-Security.md)）。
- 站点开关影响 UI：`plaza_enabled` 控制广场入口、`registration_mode` / `oidc_enabled`
  控制注册与 SSO、`about_enabled` 控制关于页。配置来自 `GET /api/v1/config`。

## 与后端交互

| 项 | 实现 |
|---|---|
| 基础路径 | `fetch('/api/v1' + path)`，统一信封 `{status,message,data}`；失败抛 `ApiError`（含 code / retryAfter） |
| 会话 | **HttpOnly Cookie `imgli_session`**，前端不存 token、不设置 `Authorization` |
| API Token | 明文仅在创建响应中显示一次，放在组件 state 的一次性 banner；片段基址取 `config.base_url` |
| 列表 | 图库 / 广场用 `useInfiniteQuery` 游标分页（limit 24）；`useSession` / `useConfig` staleTime 5min |
| 上传 | `XMLHttpRequest` 单请求 multipart，字段 `file / visibility / album_id / policy_id / expires_in / max_views`；`upload.onprogress` 计算百分比，支持 abort |
| 队列 | zustand 并发上限 **3**，状态 `queued/uploading/processing/success/instant/failed`；本地 `URL.createObjectURL` 缩略图；`rate_limited` 按 `Retry-After` 自动重试；`instant` 为秒传复用结果 |
| URL 抓取 | `POST /upload/url`，服务端拉取，无进度 |
| 实时通知 | **没有 SSE / WebSocket**。仅管理端存储迁移任务 1s 轮询；其余靠本地 toast、公告条与 PWA SW |
| localStorage | 只存偏好：`imgli-theme` / `imgli-view` / `imgli-lang` 与各类 dismissed 标记 |

主题分两层：本地 `light/dark/system`（`src/store.ts`）+ 运营级站点配色
（`src/lib/siteTheme.ts`：强调色、背景图、遮罩与毛玻璃不透明度）。
设计说明见 [docs/design/site-customization-ia.md](../docs/design/site-customization-ia.md)。

## 构建与嵌入

```text
npm run build → web/dist
  → //go:embed all:dist（web/embed.go，DistFS() 用 fs.Sub）
  → internal/server/server.go：Options.Web 为 nil 时回落 web.DistFS()
  → internal/server/web.go：/assets/* immutable 长缓存、index.html no-cache、
     SPA 回落、/s/ 与 /a/ 注入 OG/Twitter 元数据
```

- [Makefile](../Makefile)：`web` = `npm install && npm run build`，`web-ci` 仅 build（CI / GoReleaser），
  `make build = web + build-go`。
- [Dockerfile](../Dockerfile) 第一阶段用 `node:24-alpine` 构建前端，第二阶段复制
  `dist` 进 Go 构建阶段（`-tags vips`）。
- 开发代理（[web/vite.config.ts](../web/vite.config.ts)）：`/api`、`/i/`、`/t/` →
  `http://localhost:8686`。
- **前端没有 `.env*` / `VITE_*` 变量**：只用 `import.meta.env.PROD`（生产注册 SW）
  与 `import.meta.env.DEV`（缺翻译键告警），运行时配置来自 `GET /api/v1/config`。
- E2E（[web/playwright.config.ts](../web/playwright.config.ts)）：`../imgli serve`，
  端口 `8697`，临时 data 目录，`IMGLI_RATE_LIMIT_MULT=100`。

相关：[Development-and-Testing.md](Development-and-Testing.md) ·
[Architecture.md](Architecture.md) · [API-and-CLI.md](API-and-CLI.md)
