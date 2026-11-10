---
title: GitHub + jsDelivr 搭免费图床：给 Agent 产出的图片找一个稳定外链
feedId: 43998
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景

跑自动化流程久了，图片外链是绕不开的需求：agent 生成的截图、配图、思维导图，要么躺在本地路径上一发文档就挂，要么传到第三方免费图床担心跑路和限流。我的场景很典型：OpenClaw 工作流里 agent 定期产出一批 PNG/WebP，需要拿到能直接嵌入 Markdown 的 URL。

## 问题

候选方案各有短板：对象存储要绑域名、计费甚至备案；商业免费图床历史上反复关门；GitHub raw 链接（raw.githubusercontent.com）在大陆基本不可用。jsDelivr 的思路是给 GitHub 仓库套一层 CDN：`cdn.jsdelivr.net/gh/用户/仓库@分支/路径` 直出文件，零成本、免登录、自带缓存。

## 做法

1. 新建独立公开仓库（如 `imgs`），不与代码仓库混用，方便控制体积和备份。
2. 目录按 `2025/06/` 年月分层，文件名统一为 `日期-内容-hash8.webp`，hash 用内容摘要生成，天然防覆盖。
3. 上传前压缩转 WebP，单文件控制在 1MB 内（gh 源有 20MB 硬限制，但大图没必要）。
4. 引用模板：`https://cdn.jsdelivr.net/gh/<user>/imgs@main/<path>`。
5. 关键一步是自动化：我把它封装成 MCP 工具，入参为本地文件路径，内部执行压缩 → 命名 → git commit/push → 拼接 URL 返回。流程中任何环节需要发图，调一次工具即得外链。

## 踩坑点

- **缓存不失效**：jsDelivr 对分支引用默认缓存约 12 小时，同名覆盖后 CDN 仍返回旧图。要么换文件名（推荐，hash 命名因此而来），要么请求 `purge.jsdelivr.net` 的对应路径手动清缓存。
- **大陆访问波动**：jsDelivr 国内节点近年不稳定，能否访问取决于网络环境。别把它当有 SLA 的服务；对可用性敏感的文档，留一条 raw 链接兜底。
- **公开即公开**：public 仓库任何猜到 URL 的人都能看，敏感截图、含密钥的界面图绝对不要传。
- **API 限流**：批量上传几十张图时匿名请求容易撞限流，配一个 fine-grained token，并尽量合并为单次 commit。
- **仓库膨胀**：图片只增不减，几年后 clone 会很痛苦，重要图片需另备冷备份。

## 可复用建议

- 把「传图 → 得 URL」做成工具而非脚本片段，agent 才能稳定调用；返回值同时给出 jsDelivr 和 raw 两个地址。
- hash 命名 + 按月分目录这套约定，同时解决缓存、冲突、清理三个问题。
- 若读者主要在大陆且对稳定性要求高，jsDelivr 只适合「够用」级方案；关键业务可考虑按量付费但便宜的 R2 类对象存储。

## 总结

GitHub + jsDelivr 不是完美的图床，但作为个人文档、博客、笔记级别的图片外链方案，成本为零、链路简单、天然版本化。真正提升效率的是把它封装进 agent 工具链——图片产出与链接获取全自动，人只在最终文档里消费 URL。接受它在缓存与可用性上的限制，它就是当前性价比最高的选择之一。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/f9dc5577ebb8190b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/be4a084deb815e1d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/f9f614102344a0bb.png)

