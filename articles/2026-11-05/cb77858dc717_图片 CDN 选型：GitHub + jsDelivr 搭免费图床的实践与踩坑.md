---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践与踩坑
feedId: 43536
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

写博客、维护插件文档、让 Agent 产出带图的 Markdown 报告，都绕不开图片外链这个环节。商用 OSS 稳定但要钱，各家免费图床跑路和限速风险高，GitHub 原生的 `raw.githubusercontent.com` 在国内基本不可用。折中下来，**GitHub 仓库 + jsDelivr CDN** 是成本最低、可持续性尚可的组合。我把它接进了自己的自动化写作流程用了小半年，这里记录完整做法和踩过的坑。

## 问题

核心诉求有三个：链接在国内可直连；上传能被 Agent 自动完成；缓存策略可控。GitHub + jsDelivr 前两条天然满足，第三条要自己下功夫。

## 做法

1. **建一个公开仓库**，比如 `assets`，专放图片，按 `年/月` 目录组织。
2. **上传走 GitHub Contents API**：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/me/assets/contents/img/2025/0611-demo.webp \
  -d '{"message":"add img","content":"<base64>"}'
```

无鉴权限额只有 60 次/小时，带 PAT 是 5000 次/小时，token 必须配。

3. **引用换成 jsDelivr 域名**：
   `https://cdn.jsdelivr.net/gh/me/assets@main/img/2025/0611-demo.webp`
   国内可达性明显好于 raw 域名。要长期稳定引用，建议把 `@main` 换成 commit SHA 或 release tag，链接即不可变。
4. **封装成 MCP 工具**：把“本地图片 → 压缩转 WebP → base64 → API 上传 → 返回 CDN 链接”做成一个工具暴露给 Agent。写文档时直接调用，产出的 Markdown 图片链接开箱即用，人不用碰上传环节。

## 踩坑点

- **缓存不更新**：jsDelivr 对同名文件缓存很激进，覆盖上传后线上仍是旧图。解法是文件名带日期或内容 hash（如 `0611-a3f2.webp`），只增不改；紧急情况走 `purge.jsdelivr.net` 主动刷新。
- **国内可达性波动**：2022 年之后 jsDelivr 国内节点时好时坏。备好 `testingcf` / `fastly` / `gcore` 几个镜像前缀做失败切换，关键文档同时保留 raw 链接兜底。
- **体积限制**：单文件 50MB 上限，超大仓库会被限流。上传前统一压缩，单张控制在小几百 KB，仓库保持精简。
- **隐私与合规**：公开仓库任何人可见，截图和日志类图片先脱敏。jsDelivr 定位是服务开源项目文件，个人图床属灰色地带——量小、克制地用没问题，别当网盘。

## 可复用建议

- 先定命名规范：`{日期}-{hash或描述}.{ext}`，缓存问题少一半。
- 上传逻辑做成独立 MCP 工具或脚本，Agent 可调用，流程才真正自动化。
- 写一个十几行的多 CDN fallback 函数，几个域名轮询重试。
- 心态上把它当“尽力而为”的基础设施：博客、文档、Agent 产出物够用；生产关键链路请上付费 OSS。

## 总结

GitHub + jsDelivr 不是完美方案，但胜在零成本、全流程可自动化、与 Git 工作流天然契合。配好缓存策略和 fallback 之后，个人场景的稳定性完全够。它更像一块顺手的胶带，而不是承重墙——知道边界在哪，用起来才踏实。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/bc424b95186e1ba0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/5655b5a910d7db3a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/2a270922b8eabb6c.png)

