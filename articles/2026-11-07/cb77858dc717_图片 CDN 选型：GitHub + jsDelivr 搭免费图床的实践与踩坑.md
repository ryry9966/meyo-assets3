---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践与踩坑
feedId: 43692
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

写博客、维护文档、给 Agent 生成的内容配图，都需要一个稳定的图片外链。对象存储要备案和持续付费，各家免费图床额度收紧、随时可能跑路。对个人开发者来说，**GitHub 仓库做存储、jsDelivr 做全球 CDN 分发**的组合几乎是零成本方案，在 OpenClaw 的自动化流水线里也容易集成——Agent 写作时的截图、插件产出的示意图，都可以走这条路。

## 问题

核心诉求有三个：

1. 外链稳定，支持 HTTPS，最好有边缘节点缓存；
2. 能被自动化：脚本或 MCP 工具一键上传并拿回 URL；
3. 更新可控：图片改了之后，CDN 不能无限期缓存旧版本。

## 做法

1. 新建一个公开仓库（如 `assets`），按 `yyyy/mm/` 目录或哈希文件名组织图片；
2. 引用地址格式：`https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>`；
3. 自动化上传两条路：
   - **简单**：调 GitHub Contents API，一次 PUT 提交文件，返回后拼接 CDN URL；
   - **稳妥**：本地 commit + push，或封装一个 MCP 工具，内部处理"上传 → 命名 → 返回 CDN 链接"，让 Agent 在写作流程里直接调用；
4. 更新图片时，可请求 `https://purge.jsdelivr.net/gh/...` 主动刷缓存；更推荐直接换文件名，把版本烧进 URL。

## 踩坑点

- **jsDelivr 明确不鼓励当纯图床用**。其条款禁止把 CDN 当作通用文件/媒体托管服务。个人文档配图的量级一般没事，但当成相册大量存图，有被限制的风险。控制体量，原始文件永远以 GitHub 仓库为准，CDN 只是加速层。
- **缓存不可变**。同一 URL 更新文件内容后，边缘节点可能长时间返回旧图。要么 purge，要么用 commit hash 或新文件名做不可变引用。
- **国内可达性波动**。jsDelivr 的合规情况历史上反复过，直连时好时坏。`fastly.jsdelivr.net`、`gcore.jsdelivr.net` 等备选域名可以写进降级逻辑，前端用 `<img onerror>` 切换兜底。
- **限制与隐私**：单文件约 20MB 上限；公开仓库无隐私可言，截图先脱敏、大图先压缩。
- **API 限速**：批量自动化上传时注意 Contents API 限额，批量场景走 git push，不要逐个调 API。

## 可复用建议

- 把"上传图片 → 返回 URL"封装成一个 **MCP 工具**，是接入 Agent 工作流最省事的方式，命名、压缩、重试都收敛在工具内部；
- 目录与命名规范提前定好：`blog/2025/06/xxx-hash.png`，URL 本身就是索引；
- Markdown 里保留仓库相对路径的注释，CDN 万一失效，随时可从源仓库切换引用方式；
- 体量上来后（几千张、有流量压力）再迁对象存储，方案不用推倒重来——因为 URL 生成和上传逻辑已经收敛在一个工具里，换底层只改实现。

## 总结

GitHub + jsDelivr 不是企业级的"长久之计"，但对个人知识库、博客、Agent 产出物配图，是投入产出比很高的一档：零成本、纯 Git 工作流、天然版本化。关键是想清楚边界——控制用途和体量，做好缓存与降级策略，并把上传动作工具化，让底层随时可替换。缺什么补什么，不提前过度设计。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/04a9b7b8b19c9441.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/1cea93c9413fcd04.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/e1e0c8ca23d33e51.png)

