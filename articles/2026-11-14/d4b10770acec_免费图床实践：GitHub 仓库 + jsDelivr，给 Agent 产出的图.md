---
title: 免费图床实践：GitHub 仓库 + jsDelivr，给 Agent 产出的图片一个稳定 URL
feedId: 44390
source: 综合讨论
publishedAt: 2026-11-14
---

## 背景

写 OpenClaw 插件文档、跑 Agent 自动化流程时，经常需要给图片一个可外链的 URL：Agent 生成的流程图、运行截图、Markdown 报告里的插图，要贴进 README、博客或对话输出，本地路径是没用的。图需要一个稳定、免维护、最好零成本的托管 + 分发方案。

## 问题

把候选方案过一遍：

- 商业 OSS + CDN：稳定，但要开账号、计费，国内 CDN 还涉及备案，个人和小项目不值得；
- 免费图床（SM.MS 之类）：有配额和防盗链，还有服务关停的风险，图没了链接就成死链；
- GitHub raw 链接：免费、天然版本化，但 `raw.githubusercontent.com` 在国内可达性差，且有速率限制。

结论：存储和分发分开——GitHub 仓库做存储层，jsDelivr 做边缘分发层，各干各的。

## 做法

1. **建一个公开仓库**，比如 `imgs`，只放图，按年月分目录。
2. **上传**：手动批量走 git push；程序化上传不必 clone，直接调 GitHub Contents API，base64 一次 PUT 搞定：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/USER/imgs/contents/2025/06/demo.png \
  -d '{"message":"add demo.png","content":"'"$(base64 -w0 demo.png)"'"}'
```

3. **引用**，URL 模板：
   `https://cdn.jsdelivr.net/gh/USER/imgs@main/2025/06/demo.png`
4. **接入 Agent 工作流**：把「上传图片 → 返回 CDN URL」封成一个 MCP 工具（比如叫 `img_upload`，参数就一个 `local_path`）。Agent 截图后直接拿到可嵌入 Markdown 的链接，下游的渲染、发帖、归档全程自动化，几十行代码的事。

## 踩坑点

- **缓存不更新**：jsDelivr 按 URL 缓存，同名覆盖不会生效。要么文件名带日期 + 哈希，要么用 `@<commit hash>` 引用做不可变链接，要么手动请求 purge 接口（`https://purge.jsdelivr.net/gh/...`）清缓存。
- **文件限制**：单文件约 20MB 上限；仓库体积过大后 jsDelivr 可能拒绝服务。Git LFS 的文件 jsDelivr 只会返回指针文本，等于废了，图床仓库别开 LFS。
- **仓库必须公开**：任何能进仓库的截图都可能被外人看到，别带 token、内网地址、客户数据。
- **国内可用性波动**：jsDelivr 在大陆的可达性历史上起伏过，偶尔抽风。个人博客、文档、demo 够用；如果是商业产品强依赖的图片，老老实实上付费 OSS + CDN，别省这个钱。
- **别当无限量服务用**：jsDelivr 有滥用策略，大流量热点会被限速，它的定位是开源静态资源分发。

## 可复用建议

- 命名规范用 `YYYY-MM-DD-短哈希.png`，天然防重名，也利于缓存失效；
- 开发期用 `@main` 方便调试，发布后的文章/文档改用 `@<commit>`，链接永不漂移；
- 上传工具做成 MCP server，团队里谁的 Agent 都能挂，接口保持最小；
- GitHub 仓库本身是唯一事实源，jsDelivr 只是缓存层——备份和迁移策略按 GitHub 仓库来设计。

## 总结

这套方案的本质是「GitHub 做存储 + jsDelivr 做边缘」，零成本拿到版本化、可外链的图片服务，再配一个极小的 MCP 工具就能无缝进 Agent 流程。它适合文档、博客、demo 和个人自动化；不适合有强 SLA 要求的生产场景。想清楚边界，用得就踏实。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/126850518154e4c8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/4ae3e54b7e823dfe.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/da324bb486154679.png)

