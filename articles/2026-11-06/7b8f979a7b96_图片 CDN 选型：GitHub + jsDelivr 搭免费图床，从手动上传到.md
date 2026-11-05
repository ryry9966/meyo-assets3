---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床，从手动上传到 Agent 可调用
feedId: 43583
source: 综合讨论
publishedAt: 2026-11-06
---

## 背景

写博客、发文档、让 Agent 输出带图报告时经常要贴图。本地路径随文章迁移就失效，对象存储要花钱还可能要备案，免费图床这几关停的不少。反复权衡后我落地了一个最朴素的方案：GitHub 公共仓库当存储，jsDelivr 当 CDN。零成本、零服务器、链接规则透明，且天然适合自动化。

## 前提认知

jsDelivr 的 `gh` 源会缓存 GitHub 公共仓库的静态文件。所以两个硬性前提：仓库必须是 public；不要直接引用 `raw.githubusercontent.com`（大陆访问慢且不稳定），jsDelivr 相当于套了一层全球缓存。

## 做法

**1. 建仓库与目录规范**

建一个公共仓库如 `imgs`，按用途分目录：`blog/`、`agent-output/`、`screenshots/`。只放图片。

**2. 链接规则**

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<分支或commit>/<路径>
```

例如 `https://cdn.jsdelivr.net/gh/me/imgs@main/blog/2025/a.png`。

**3. 上传自动化（对 OpenClaw 用户的关键一步）**

手动上传网页拖拽即可，但价值在接入自动化：写个几十行的脚本，调 GitHub Contents API PUT 文件，拿到 commit 后拼出 jsDelivr URL 返回给 Agent。再把这个脚本封装成一个小 CLI 或 MCP tool，输入本地路径、输出 CDN URL——图片上传从此变成 Agent 可随时组合调用的原语。

**4. 缓存刷新**

jsDelivr 对分支引用默认缓存约 12 小时。替换同名文件想立即生效，访问 `https://purge.jsdelivr.net/gh/user/repo@main/路径` 手动 purge。

## 踩坑点

- **大陆访问时好时坏**。jsDelivr 2022 年后 ICP 状态变化，线路不完全稳定。面向大陆读者的严肃场景要接受偶发不可达，或另备一条回源路径。
- **仓库不是网盘**。单文件超 100MB 推不上去，仓库过大 jsDelivr 可能拒绝服务。只放图片，总量控制在 1–2GB 内，别拿来分发安装包和视频。
- **同名覆盖不会立刻生效**（缓存）。推荐彻底避免覆盖：文件名用 `日期-内容短hash.扩展名`，天然去重、可回溯。
- **public 即公开**。截图里的 token、聊天记录、内网地址必须先脱敏。我在上传脚本里加了一道正则清洗，Agent 传图时自动执行。
- **API 有限流**。批量上传时串行执行、失败重试，别打爆配额。

## 可复用建议

- 上传逻辑统一收敛到一个工具入口，Agent 不直接碰 API，方便换实现。
- 文章里不要写死 CDN 域名，用配置变量管理，将来迁移只改一处。
- 每次上传把 URL 和原始文件名追加进同仓库的 manifest 文件，删改有据可查。
- 涉密、含个人信息的图一律不走这条链路。

## 总结

GitHub + jsDelivr 不是性能最优的 CDN，但在"零成本、规则简单、可自动化"三个约束下是最均衡的选择。对 OpenClaw 用户的真正收益不在省带宽，而在把图床做成 Agent 工具链里一个可复用的能力。风险点集中在大陆访问稳定性，用途轻量、命名去重、做好兜底，这套方案可以长期服役。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/2fa9f6e67802d7ab.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/c7ae49ca00392410.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/c1d23aa88240821f.png)

