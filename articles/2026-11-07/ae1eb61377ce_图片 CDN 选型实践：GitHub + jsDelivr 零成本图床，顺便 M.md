---
title: 图片 CDN 选型实践：GitHub + jsDelivr 零成本图床，顺便 MCP 化
feedId: 43752
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

最近在给 OpenClaw 的 agent 做自动化输出：生成的周报要贴截图，博客写作流里配图频繁，插件文档也常年缺图。图床是绕不开的基础设施。主流选项我都过了一遍：对象存储要实名 + 备案，免费额度用完就收费；SM.MS 免费档有容量和防盗链限制；自建 Vercel/Cloudflare 方案配置成本不低。最后落在一个老办法上：**GitHub 公开仓库 + jsDelivr CDN**——零成本，且天然适合被脚本和 agent 调用。

## 诉求与选型逻辑

我的硬性条件只有三条：

1. 免费，量大了也不心疼；
2. 图片 URL 稳定，国内至少有可用的降级路径；
3. 能通过 API 自动上传，方便封装成 MCP tool。

分工很清晰：GitHub 仓库是存储层和唯一数据源，jsDelivr 是缓存层。缓存层坏了随时可换，数据不丢。

## 做法

**第一步**：建一个公开仓库（如 `imgs`）。私有仓库 jsDelivr 不认。

**第二步**：确认 URL 规则：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<ref>/<path>
```

`@main` 跟随最新提交；`@v1.0` 这类 tag 会被永久缓存。

**第三步**：上传自动化。核心是 GitHub Contents API：`PUT /repos/{owner}/{repo}/contents/{path}`，body 放 base64 内容和 commit message。用 fine-grained PAT，只授予这一个仓库的 Contents 读写权限。我把这段逻辑封装成了 MCP tool：agent 产出图片后直接调用，返回 CDN URL，写作流里不再人工搬图。

**第四步**：缓存策略。采用 append-only 命名——文件名带 8 位内容 hash，如 `2025/06/a3f9c1d2-shot.png`。永远新增、不覆盖，从根上绕开 CDN 缓存刷新问题；确需刷新时用 `purge.jsdelivr.net` 手动 purge。

## 踩坑点

- **国内直连不稳是最大变数**。`cdn.jsdelivr.net` 时好时坏，备用域名 `fastly.jsdelivr.net`、`gcore.jsdelivr.net`、`testingcf.jsdelivr.net` 实测可互为替补；再不行就 Cloudflare Workers 反代（记得绑自定义域名）。
- **同名覆盖后 CDN 大概率不更新**。别赌缓存刷新，直接换文件名，这也是 append-only 的由来。
- **仓库别当网盘**。单文件压到 5MB 以内，仓库总体积控制在 GB 级以下。jsDelivr 的定位是开源项目 CDN，纯图床属于灰色用法，量级要有分寸。
- **PAT 用 fine-grained 最小授权**。一旦泄漏，爆炸半径只有这一个仓库。

## 可复用建议

1. 数据库里只存 `user/repo@ref/path`，不要存完整 jsDelivr URL。哪天换域名、换 CDN，全局替换一次字符串即可，无需迁移。
2. 上传前先压缩（pngquant / squoosh），省仓库体积也省流量。
3. 挂一个 GitHub Action 定期做图片无损压缩（如 calibreapp/image-actions），防止协作者传大图。
4. 把 `image_upload` 做成通用 MCP tool，agent 写文档、发周报、存截图都能复用同一入口。

## 总结

这套方案的本质是**把开源 CDN 当缓存层用，存储层和缓存层彻底解耦**。它不完美——国内可用性是持续的赌注，重度滥用还有被限制的风险——但零成本、API 友好、数据主权在自己手里，作为个人项目和 agent 自动化场景的图床完全够用。CDN 随时可以换，数据永远是自己的，这是选它而不是纯第三方图床的根本原因。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/29cfe770a51b96c9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/bcf839434ed04927.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/afa5a8d4a4ea21fe.png)

