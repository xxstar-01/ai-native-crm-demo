# 静态演示站部署说明

这个目录就是可发布的站点根目录：`index.html` 在根上，全部资源用相对路径，
**hash 路由**（`#/app/ai/workbench`），因此不需要任何服务端 rewrite / SPA fallback。
纯静态、无外部 CDN、无网络请求、无构建步骤。

- 体积：约 370 KB（14 个资源文件 + 404）
- 入口：`/#/app/ai/workbench`（`/` 也会落到同一个应用）
- 已在 `_headers` / `vercel.json` / `netlify.toml` 里放了 CSP、nosniff、缓存策略

---

## 方案对比（按省事程度排序）

| 渠道 | 免费额度 | 国内访问 | 自定义域名 | 备案 | 部署方式 |
|---|---|---|---|---|---|
| Cloudflare Pages | 免费，带宽不限 | 一般（可自定义域名 + 优选 IP 改善） | 支持 | 不需要 | 拖拽 / `wrangler` / Git |
| 腾讯云 EdgeOne Pages | 有免费套餐 | 好（国内节点） | 支持 | 国内加速需备案 | 控制台导入 / Git / CLI |
| Netlify | 免费额度够用 | 一般 | 支持 | 不需要 | 拖拽 / CLI / Git |
| GitHub Pages | 免费（1GB 仓库，软限 100GB/月） | 偏慢但通常可用 | 支持 | 不需要 | 推仓库 + 设置 Pages |
| Vercel | Hobby 免费 | `*.vercel.app` 在国内常不可达 | 支持 | 不需要 | `vercel` CLI / Git |
| 阿里云 OSS / 腾讯云 COS + CDN | 按量计费，很便宜 | 最好 | 支持 | 国内节点需备案 | `ossutil` / `coscli` + CDN |

结论建议：
- 主要给国内面试官/HR 看 → **EdgeOne Pages**（快）或 **Cloudflare Pages**（免备案、省事）。
- 只要一个能点开的链接、不折腾 → **Netlify Drop** 拖拽，30 秒。
- 已经有 GitHub 仓库 → **GitHub Pages**，顺手；再加 Cloudflare 代理可提速。
- 正式长期用、有自己的域名和备案 → **对象存储 + CDN**。

---

## 1. Netlify Drop（最快，无需命令行）

1. 打开 <https://app.netlify.com/drop>，登录。
2. 把**这个目录**整个拖进去（拖目录，不是拖 zip）。
3. 得到 `https://<随机名>.netlify.app`；在 Site settings 里可改子域名 / 绑自定义域名。

或命令行：

```bash
npm i -g netlify-cli
netlify deploy --prod --dir .        # 在当前目录执行，首次会引导登录和建站
```

## 2. Cloudflare Pages（推荐：免备案 + 全球 CDN）

控制台方式：Workers & Pages → Create → Pages → **Upload assets** → 拖入本目录 → Deploy。

命令行方式：

```bash
npm i -g wrangler
wrangler login
wrangler pages deploy . --project-name ai-native-crm-demo
```

（目录里的 `wrangler.toml` 已写好 `pages_build_output_dir = "."`，也可以直接
`wrangler pages deploy`。）

绑定自定义域名：Pages 项目 → Custom domains → 添加，按提示加 CNAME。
国内速度一般，可用「自定义域名 + Cloudflare 优选 IP」的常见做法改善，但别指望和国内 CDN 一样。

## 3. 腾讯云 EdgeOne Pages（国内访问最好）

1. 控制台 → EdgeOne → Pages → 新建项目 → 选择「直接上传」或导入 Git 仓库。
2. 上传本目录（构建命令留空，输出目录 `/`）。
3. 默认会给你一个 `*.edgeone.app` 域名；绑自定义域名时，**国内加速需要域名已完成 ICP 备案**。

## 4. Vercel（海外访问最顺，国内慎用）

```bash
npm i -g vercel
vercel --prod            # 目录内执行，Framework 选 Other，Build Command 留空，Output 设为 .
```

国内直连 `*.vercel.app` 经常超时，适合发给海外或能科学上网的人。

## 5. GitHub Pages（已有仓库时最顺手）

```bash
# 在仓库根目录，把本目录内容推到 gh-pages 分支
git subtree push --prefix examples/chat-first-demo/dist origin gh-pages
# 或手动：把本目录内容复制到一个 orphan 分支再 push
```

仓库 Settings → Pages → Source 选 `gh-pages` / `main` + `/docs`。
项目页地址形如 `https://<user>.github.io/<repo>/`——hash 路由 + 相对路径，放在子路径下也能正常工作。
目录里已有 `.nojekyll` 和 `404.html`。

## 6. 阿里云 OSS / 腾讯云 COS（正式、可控）

```bash
# 阿里云
ossutil cp -r . oss://<bucket>/ --update
# 腾讯云
coscli cp . cos://<bucket>/ -r
```

存储桶开启「静态网站托管」，默认首页 `index.html`、错误页 `404.html`。
注意：直接用 OSS/COS 默认域名访问 HTML 常被强制下载，**要正常浏览需绑定自定义域名**，
而国内节点的自定义域名需要备案。

---

## 临时演示（不部署，几分钟给个链接）

本机起服务后开隧道，适合面试前临时发链接，电脑关了即失效：

```bash
# 本机
node scripts/serve-agent-demo.mjs          # http://127.0.0.1:3008

# 另开一个终端（Cloudflare 快速隧道，无需账号）
cloudflared tunnel --url http://127.0.0.1:3008
```

会得到 `https://<随机>.trycloudflare.com`。ngrok 同理（需免费账号的 token）。

---

## 部署后自检

1. 打开 `https://<域名>/#/app/ai/workbench`，确认侧边栏、对话、能力实验室都正常。
2. 硬刷新一次，确认没有 404（Network 里 14 个资源全 200）。
3. 手机宽度看一眼（侧边栏折叠、抽屉）。
4. 控制台应无报错；`connect-src 'none'` 下不应出现任何网络请求。

## 更新站点

内容变了就重新上传/重新 deploy 同一目录即可；`index.html` 是 `max-age=0`，
JS/CSS 是长缓存 + `immutable`。**改了 JS/CSS 但文件名没变时**，老访客可能拿到旧缓存——
这个项目没有构建哈希，改完可以顺手把 `_headers`/`vercel.json` 里的 `immutable` 去掉，
或者把文件名加个版本后缀。
