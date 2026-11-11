---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践与边界
feedId: 44130
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景

写技术文档、社区帖，或让 Agent 产出带图内容，都绕不开同一个问题：图片放哪。塞进项目仓库会让 git 历史越来越肥；直接贴 raw.githubusercontent.com 外链，不少网络环境下慢甚至超时，而且 raw 不缓存、有并发限制。社区文档流程里我们的做法是：GitHub 公开仓库当存储，jsDelivr 当分发缓存层，再把「传图、拿外链」封装成 MCP 工具，供写帖脚本和 Agent 调用。

## 方案与步骤

1. 建一个公开仓库（如 `cdn-images`），按用途分目录：`blog/`、`docs/`、`bot/`，别散建多仓。
2. 引用一律带版本：`https://cdn.jsdelivr.net/gh/<user>/<repo>@<commit-hash>/blog/2025/05/demo.webp`；不要裸用 `@main`，原因见下。
3. 上传两条路：手动是本地转 webp 压缩后 `git push`；自动化走 GitHub Contents API（PUT 即 commit），或 GitHub Actions 监听目录同步。
4. 封装成 MCP 工具或 OpenClaw 插件：输入本地路径或截图，返回可直接粘贴的外链，Agent 写帖即取即用。
5. 异常时逐条请求 `https://purge.jsdelivr.net/gh/...` 强刷缓存；大文件走 GitHub Releases，`@version` 同样能引用附件。

## 踩坑点

- **别指望国内加速。** jsDelivr 大陆节点自 2022 年起持续不稳，官方也不再承诺大陆可用。它在海外和服务器侧很稳，国内读者打开慢是常态；要服务国内，直接上 Cloudflare R2 绑自有域或国内对象存储。
- **缓存跟版本走。** jsDelivr 对分支引用缓存很久，`@main` 下改图常半天不生效；用 commit hash 或 tag 引用，改图即新 URL，免去失效焦虑。
- **别当网盘。** jsDelivr 定位是开源项目静态资源分发，滥用的大仓库会被限流甚至 403。控制总量，单图压到几百 KB。
- **公开即公开。** 仓库所有人可见，git 历史里的图删不干净，隐私截图一律不进库。
- **新图可能 404。** 刚 push 完外链未必立即可用，purge 一下或等片刻；自动化流程里加一次 HEAD 重试。
- **raw 链接别外发。** raw.githubusercontent.com 不缓存、有限流，只当上传通道，别当展示链接。

## 可复用建议

- URL 模板固定：`gh/<user>/<repo>@<hash>/<用途>/<日期>/<语义化文件名>`，便于检索和批量替换。
- 上传前统一转 webp 并压缩，PNG 截图通常能从 MB 级降到 100KB 上下。
- 把「压缩→上传→拿外链→HEAD 校验」做成一个 MCP tool，是收益最大的部分：人不用管细节，Agent 也能正确带图。
- 文档保留完整 hash URL，换仓库或迁 CDN 时可按映射脚本批量重建。
- 对存量外链定期 HEAD 抽查；jsDelivr 对个别仓库的封禁常是事后发生，早发现早迁移。

## 总结

这套方案本质是「GitHub 当存储、jsDelivr 当缓存层」：零成本、URL 稳定、天然版本化，对个人博客和社区文档够用。边界也清楚——不解决国内加速，扛不住大流量，还有第三方服务的政策风险。把它当低频静态资源的分发层并自动化上传流程，性价比很高；一旦图片要面向国内读者或流量上量，直接换对象存储更省心。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/9deae7bbc29bdf3d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/112ed0960ba9f044.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/2ef9469d875bd669.png)

