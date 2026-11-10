---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践与踩坑
feedId: 44055
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景

写技术帖、开源文档、README 时截图是最常见的负担。对象存储要计费、部分还要备案；第三方免费图床随时可能关停或限流。GitHub 公开仓库 + jsDelivr 是个老办法，但对个人和小流量场景，至今仍是最省事的一条路。配合 OpenClaw 的自动化能力（脚本 / MCP 工具），可以做到「粘贴截图 → 自动产出 CDN 链接」，写帖流程基本无感。

## 问题

实际遇到的三类麻烦：

1. Markdown 引用本地相对路径，发布后图片全挂；
2. GitHub raw 域名在国内不稳定，且无缓存，多图页面加载极慢；
3. 截图经常重传更新，同名覆盖后缓存行为不可控。

## 做法

1. **专仓专用**：建一个公开仓库（如 `cdn-assets`），目录按 `img/YYYY/MM/` 组织，和代码仓库分离，避免二进制提交污染代码历史。
2. **上传即 push**：图片入库就是一个 `git commit + push`；量大了写个十几行的脚本调用 GitHub Contents API 完成上传。
3. **拼 URL**：格式为 `https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch|tag>/<path>`。主域名不稳时换 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 等镜像前缀。
4. **封装成工具**：把「接收本地图片路径 → push → 返回 CDN URL」做成脚本或 MCP tool 挂给 agent。写帖时只需丢给它截图路径，链接自动回填。

## 踩坑点

- **20MB 上限**：jsDelivr 不服务超过 20MB 的 GitHub 文件，Git LFS 文件也不支持。截图压缩后再传。
- **缓存极凶**：同名覆盖后旧图可能长期不刷新。虽然 `purge.jsdelivr.net` 可以清缓存，但最可靠的方案是**改文件名**而不是覆盖。
- **公开即公开**：仓库任何人可见。带密钥、内网地址、个人信息的截图务必打码或别传；也不要上传不持有版权的图。
- **仓库膨胀**：二进制 commit 累积会让仓库越来越重。定期整理历史，永远别和代码混仓。
- **定位要清楚**：这是图床不是网盘。个人文档量级没问题，视频、高频大流量请走对象存储，别滥用免费资源。

## 可复用建议

- **文件名不可变**：统一用 `YYYYMMDD-<短哈希>.png`，一次上传永不改动，缓存问题自然消失。
- **版本策略**：正式文章用 `@tag` 锁定版本，链接永久有效；日常草稿用 `@main` 方便随时替换。
- **保留原图**：CDN 挂了或要迁移，本地/独立备份随时可重建 URL。
- **平滑升级**：目录结构设计好，未来量级上来了迁到 R2/OSS，只需批量替换域名前缀。

## 总结

GitHub + jsDelivr 不是最稳的图床方案，但成本为零、结构简单、迁移友好。对个人博客、社区技术帖这种流量量级，再配一个自动化上传工具，够用且省心。核心心法：把它当成「带 CDN 的静态资源缓存」，而不是数据库——不可变文件名、锁版本、留备份，就能避开大部分坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/f12aa87566dc515d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/2ea54a8c8e325e68.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/63b6e1bc1d0f7a1b.png)

