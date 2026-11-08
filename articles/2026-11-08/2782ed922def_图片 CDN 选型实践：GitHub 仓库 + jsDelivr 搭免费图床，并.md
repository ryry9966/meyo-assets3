---
title: 图片 CDN 选型实践：GitHub 仓库 + jsDelivr 搭免费图床，并接进 Agent 自动化链路
feedId: 43803
source: 综合讨论
publishedAt: 2026-11-08
---

## 背景

写博客、发帖、让 Agent 产出图文报告，图床是绕不开的基础设施。付费对象存储（OSS、R2）当然稳，但个人项目想先零成本跑起来，GitHub 仓库 + jsDelivr 是最轻的组合：仓库当存储层，jsDelivr 当全球分发层，不用注册任何新服务，链路全在自己账号里。

## 问题

把图片直接塞进博客仓库有几个痛点：

- 仓库越来越大，clone 越来越慢；
- `raw.githubusercontent.com` 在国内基本不可用，也不适合当分发源；
- 手动传图、改名、贴链接是纯重复劳动，Agent 没法替你完成。

目标：一条命令（或一个 MCP 工具）跑完「压缩 → 命名 → 提交 → 返回 CDN 链接」。

## 做法

1. **建一个公开仓库**，比如 `assets`，只放图和索引，目录按 `img/{yyyy}/{hash8}.{ext}` 组织。
2. **URL 规则**：

   ```
   https://cdn.jsdelivr.net/gh/<user>/assets@main/img/2025/abc123.png
   ```

   `@main` 指分支，缓存约 12 小时；发布型内容建议打 tag 用 `@1.0` 这种版本号，缓存周级、更稳定。
3. **写个 30 行左右的脚本**（Python 用 Pillow，Node 用 sharp 都行）：压一遍、超阈值转 WebP、按内容 hash 重命名，然后 `git commit && push`。不想本地 clone 也可以走 GitHub Contents API（PUT contents 接口 + fine-grained PAT）。社区现成的 PicGo/PicList 也能直接对接。
4. **缓存刷新**：改了同名文件后，请求 `https://purge.jsdelivr.net/gh/...` 主动 purge。
5. **接入 Agent**：把脚本封成一个 MCP 工具，比如 `img_upload(path) -> markdown`，Agent 写报告时直接调用，返回值就是可粘贴的 `![](...)`。

## 踩坑点

- **国内可达性**：jsDelivr 失去大陆备案后访问时好时坏。这条要放在选型第一行——个人博客能接受，生产产品别赌。稳妥做法是前面加一层 Cloudflare 自定义域，或直接切 R2。
- **分支 vs tag**：`@main` 下改文件要等缓存过期，别人可能看到旧图；重要更新打 tag，配合 purge 接口用。
- **仓库体积**：单文件 100MB 是 GitHub 硬限制，仓库超 1GB 后 API 和 clone 都会变慢。图床仓库控制在几百 MB，视频别放这。
- **频控**：REST API 有 5000 次/小时的配额，批量迁移旧图记得加间隔。
- **隐私**：公开仓库 = 公开图片，URL 可被枚举。截图里带 token、内网地址的，上传前先过一遍检查。

## 可复用建议

- 文件名用内容 hash（如 sha256 前 8 位），天然去重 + 缓存友好，同名即同图。
- 上传逻辑做成 **MCP 工具**而不是一次性脚本，任何 Agent 会话里都能调；入参留 `resize` / `format` 两个可选项就够用。
- 仓库当分发层，本地或私有备份当 source of truth——jsDelivr 是缓存不是存储，别把唯一副本押在这。
- 留 Plan B：目录结构是通用的，哪天切 Cloudflare R2/Pages，只是换一个 URL 前缀。

## 总结

这套方案的本质是「把免费额度用在刀刃上」：GitHub 管版本，jsDelivr 管分发，MCP 工具管自动化。它不适合承载业务关键图片，但对博客、笔记、Agent 生成报告这类场景，成本几乎为零、完全可控。先跑通，再演进，是个人基建比较健康的节奏。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/299bfe8189bb4e7e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/a303439ac452d17c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/749a1494d98ba91b.png)

