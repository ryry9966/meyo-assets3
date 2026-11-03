---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的实践
feedId: 43375
source: 综合讨论
publishedAt: 2026-11-03
---

## 背景

Agent 的产物里经常带图：模型生成的架构示意、运行截图、博客配图，写进 Markdown 后总得有个能外链的 URL。本地路径没法分享，商业对象存储要钱有的还要备案，临时公共图床又随时可能跑路。折腾过几轮之后，我把方案收敛到：**图片进 GitHub 仓库，jsDelivr 做 CDN 前缀**。零成本、自带版本化、API 可自动化。

## 问题

选型时我给自己定了三个硬约束：

1. 自动化要能打通——在 OpenClaw 的 skill / MCP tool 里一次调用完成「压缩 → 上传 → 返回外链」；
2. URL 要稳定可缓存——同一张图被反复引用时不能重新生成；
3. 随时可迁移——理想情况只是换个 URL 前缀的事。

## 做法

1. **建专用仓库**：新建一个公开仓库如 `cdn-assets`，只放图，和代码仓库分离。
2. **最小权限 Token**：申请 Fine-grained PAT，只授权这一个仓库的 Contents 读写。
3. **上传走 Contents API**：`PUT /repos/{owner}/{repo}/contents/{path}`，body 放 base64，curl 十几行搞定。
4. **拼外链**：`https://cdn.jsdelivr.net/gh/{owner}/{repo}@main/{path}`；备选前缀 `fastly.jsdelivr.net`、`testingcf.jsdelivr.net`。
5. **封装成 MCP tool**：入参为本地图片路径，内部先用 sharp/pillow 压一遍（长边 1600、质量 80），文件名取「日期 + 内容哈希」，上传成功后直接返回 jsDelivr URL。此后 Agent 写 Markdown 引用图片就是一次工具调用。

## 踩坑点

1. **同名覆盖不生效**。jsDelivr 缓存很激进，覆盖同名文件后 CDN 依旧回旧图。解法是把文件名设计成不可变的：`2024-06-01/a3f9c2.png`，内容变则哈希变，天然绕开缓存失效问题。
2. **国内访问波动**。jsDelivr 的 ICP 在 2022 年被吊销后，国内节点时好时坏。务必把 CDN 前缀做成配置项，出问题一键切 `fastly.jsdelivr.net`，或退回 `raw.githubusercontent.com`（海外侧稳定）。
3. **别当网盘用**。单图压到 1MB 以内，仓库总体积控制在 1GB 上下；超限后 push 变慢、API 也更容易受限，视频和大文件别放这。
4. **文件名只用 ASCII**。带中文或空格的路径拼进 CDN URL 会触发转义，某些编辑器还会二次编码导致 404。
5. **Token 暴露面**。自动化场景 Token 常驻环境，坚持 Fine-grained + 单仓库 + 仅 Contents 读写；Agent 日志里别把 Authorization 头打出来。
6. **首次生效延迟**。新文件一般几秒可用，但偶发要等回源。上传后先 HEAD 校验再落链接，比直接信返回值稳。

## 可复用建议

- 命名规范固化为 `{日期}/{哈希}.{ext}`，去重和缓存两个问题一起解决。
- 上传器写成独立小脚本（几十行），MCP tool、CI、浏览器扩展都能复用。
- URL 生成与上传解耦，前缀集中配置，迁移只改一处。
- Markdown 里用 HTML 注释保留原始 GitHub URL，CDN 抽风时可手动兜底。

## 总结

这套方案的本质是「用 Git 管静态资源，用免费 CDN 做加速」。对个人博客、Agent 产出文档配图这类**低频写、高频读**的场景，成本为零且足够稳；上限也明确——不适合大流量商用与高频写。把命名、压缩、前缀配置三件事做规范，剩下的就是一次工具调用的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-03/d7efe5b1b30c0691.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-03/37077a1f592d9e1e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-03/3b68e54c4090274d.png)

