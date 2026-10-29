# SaveTWT Cloudflare Pages 修复版

## 项目结构

- `index.html`：前端页面，提交到本站 `/api/parse`，仅在后端真实返回媒体 URL 时显示下载入口。
- `functions/api/parse.js`：Cloudflare Pages Function 后端，识别平台、调用解析服务并规范化结果。
- `manifest.json` 与 `assets/icon.svg`：PWA 基础信息。
- `_routes.json`：限制 Function 只处理 `/api/*`，静态页面仍由 Pages 直接提供。

## 当前解析方式与限制

- 抖音：调用 `https://www.tikwm.com/api/?url=...`，读取其 `code`, `data.hdplay`, `data.play` 字段。
- X/Twitter：调用 FxTwitter 的公开 JSON API，尝试读取帖子 `status.media.videos` 中的媒体 URL/variants。
- 这些都是第三方服务，本站不控制其可用性、限流、响应字段或平台兼容性。项目不会在解析失败时生成虚假下载链接。
- FxTwitter 的响应结构可能调整；如果接口未返回可识别的视频 URL，页面会明确显示失败原因。
- 只处理公开、可访问的帖子；私密、删除、登录受限或地区限制的内容可能无法解析。
- `download` 通常会打开媒体源地址。浏览器能否直接保存文件由媒体服务器的跨域和响应头决定。

## Cloudflare Pages 部署（推荐：连接 Git 仓库）

Pages Functions 需要随项目目录一起部署。最稳妥的方式是将整个项目上传到 GitHub，再在 Cloudflare Pages 中连接仓库。仅上传单个 `index.html` 不会部署 `functions/api/parse.js`。

1. 解压 ZIP，确认目录中直接包含 `index.html`、`functions/`、`_routes.json`。
2. 将这些文件提交到 GitHub 仓库根目录。
3. Cloudflare Dashboard → Workers & Pages → Create → Pages → Connect to Git。
4. 选择仓库。纯静态项目通常不需要构建命令；若 Cloudflare 界面要求输出目录，设置为项目根目录 `.`（确保根目录是 `index.html` 所在位置）。
5. 部署成功后，打开 `https://你的项目.pages.dev/`。
6. 在 `https://你的项目.pages.dev/api/parse` 直接用浏览器访问会显示方法不支持或错误响应，这是正常的；此端点需要 POST JSON 请求。
7. 先测试一条公开抖音视频，再测试一条公开 X 帖子。失败时记录页面给出的错误，或在浏览器 F12 → Network 查看 `/api/parse` 响应。

## 本地测试（可选）

需要 Node.js 和 Wrangler：

```bash
npm install -g wrangler
wrangler pages dev .
```

打开 Wrangler 显示的本地地址。注意 Pages Functions 要从项目目录加载，若 CLI 版本行为不同，请参考 Cloudflare Pages Functions 文档。

## 故障排查

- `/api/parse` 返回 404：Functions 目录没有随部署发布，或 Pages 项目输出目录选错。
- 返回 502 且提示第三方接口失败：第三方服务超时、限流、改版，或目标帖子不可访问。
- 抖音链接被识别成不支持：确认链接主机名为 `douyin.com` 或 `iesdouyin.com`；短链接重定向由服务端/解析 API 处理，若短链域名不被识别，先在浏览器中打开并复制最终链接。
- X 返回“没有提供视频直链”：该 FxTwitter 响应可能没有暴露可用的视频地址，或媒体字段结构已变更。

## 安全与合规

后端只请求写死的第三方 API，不会把用户输入 URL 当作任意服务器地址抓取，避免把此端点用作通用 SSRF 代理。不要用它绕过登录、访问控制、版权限制或下载无权保存的内容。
