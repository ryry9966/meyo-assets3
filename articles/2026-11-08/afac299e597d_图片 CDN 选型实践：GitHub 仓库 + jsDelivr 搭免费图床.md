---
title: 图片 CDN 选型实践：GitHub 仓库 + jsDelivr 搭免费图床
feedId: 43821
source: 综合讨论
publishedAt: 2026-11-08
---

## 背景

写技术帖、Agent 输出报告、README 时经常要贴截图和配图。可选方案里，对象存储要实名备案加计费，第三方免费图床随时可能跑路，自建图床需要一台有公网 IP 的机器。对 OpenClaw 这类经常自动化产出图文内容的用户来说，核心诉求其实是：**上传可全脚本化、外链稳定、内容有备份、零维护成本**。GitHub 仓库 + jsDelivr 正好把这几件事都覆盖了。

## 做法

1. **专库专用**：建一个公开仓库，比如 `cdn-images`，目录按用途分层：`/blog`、`/agent-output`、`/screenshot`。
2. **上传**：本地走 `git push`，自动化场景走 GitHub Contents API（带 token），一次 PUT 就能提交单文件，不用拉仓库。
3. **拼外链**，格式固定：

```bash
https://cdn.jsdelivr.net/gh/<user>/<repo>@main/blog/2025-06-01-a1b2c3d4.png
# 备用域
https://fastly.jsdelivr.net/gh/<user>/<repo>@main/...
# 覆盖同名文件后强制刷新缓存
curl -X POST https://purge.jsdelivr.net/gh/<user>/<repo>@main/blog/xxx.png
```

4. **封装成工具**：把「压缩 → 按 `{date}-{sha8}-{slug}` 改名 → 提交 → 拼接 URL」封装成一个 MCP 工具或 CLI。输入本地图片路径，输出 CDN 链接，Agent 一条指令完成配图。压缩用 sharp/pillow，单图控制在 1MB 内。

## 踩坑点

1. **国内连通性**：`cdn.jsdelivr.net` 直连时好时坏（ICP 历史问题），备用域 `fastly.jsdelivr.net`、`testingcf.jsdelivr.net`、`gcore.jsdelivr.net` 可兜底。入口域名务必做成配置项，不要硬编码。
2. **缓存很激进**：覆盖同名文件后 CDN 大概率仍返回旧内容。要么文件名天然唯一（推荐，带 hash），要么手动调 purge 接口。
3. **单文件 20MB 上限**，大仓库推送也慢，截图必须先压缩。
4. **必须公开仓库**，private 不通。中文文件名会被 URL 编码，各种诡异问题，一律英文命名。
5. **定位问题**：jsDelivr 是公共 CDN，不是给你当大流量网盘的。量大或商用要评估 ToS，别把它当唯一依赖。

## 可复用建议

- **仓库是 source of truth，CDN 只是缓存层**：哪天换域名、换 CDN，内容一行不动。
- 命名规范 `{date}-{sha8}-{slug}.png`，天然防缓存冲突，也方便按日期排查。
- 仓库 README 自动生成图片索引，方便人肉查找存量资源。
- 封装成 MCP tool 后，OpenClaw 的截图、报告流程可以全自动出图，这是这套方案对 Agent 用户最大的价值。

## 总结

这套方案本质是「GitHub 当存储 + jsDelivr 当缓存」：零成本、纯脚本可操作、数据在自己仓库里随时可迁移。代价是国内连通性需要多域名兜底、缓存策略要用文件名规避。对个人博客、社区帖子、Agent 产出的中小流量场景完全够用；真要规模化，再迁对象存储不迟——反正迁移成本只是换个 URL 前缀。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/75fe4ebebe141085.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/624aaac1b79b0012.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/f575feec608cab03.png)

