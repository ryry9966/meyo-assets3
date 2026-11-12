---
title: 零成本图床实践：GitHub 仓库 + jsDelivr 当图片 CDN
feedId: 44266
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

社区帖、技术文档、Agent 产出的报告里经常要贴图。我对图床的诉求很朴素：零成本、URL 稳定、国内可达、能被脚本或 MCP 工具自动调用。围绕这四条，把常见方案过了一遍。

## 问题

- **对象存储**（OSS/COS/七牛）：要实名认证，配自定义域名还涉及备案，有流量费用，个人场景太重；
- **免费图床**（SM.MS 之类）：匿名额度小、链接易失效、不可控，不适合长期引用；
- **`raw.githubusercontent.com`**：不是 CDN，无边缘缓存，部分地区加载慢，还容易被节流；
- **GitHub Pages**：带宽软限制，拿来分发图片不合适。

最后选了 GitHub public 仓库当存储、jsDelivr 的 `/gh/` 端点当 CDN，组合起来刚好满足开头四条诉求。

## 做法

1. 建 public 仓库（如 `cdn-img`），目录按 `2025/06/` 分月组织，避免单目录过大。
2. URL 规则：`https://cdn.jsdelivr.net/gh/<user>/<repo>@<ref>/<path>`。`ref` 用分支名则内容更新有缓存延迟；用 tag 或 commit 则 URL 不可变。
3. 手动上传用 PicGo（原生支持 GitHub 图床），自定义域名前缀直接填 jsDelivr 地址即可。
4. 自动化链路：写了一个 MCP 工具 `upload_image`，流程是：本地路径/base64 输入 → sharp 压成 webp（宽 ≤1600、质量 80）→ 调 Contents API PUT 提交 → 返回 jsDelivr URL。文件名用内容 hash 前 8 位 + 时间戳，天然去重防冲突。

```bash
PUT https://api.github.com/repos/<user>/<repo>/contents/2025/06/ab12cd34.webp
Authorization: Bearer <PAT>
{"message":"upload","content":"<base64>"}
```

5. 分支引用场景下，更新后通过 `purge.jsdelivr.net` 请求同一路径刷新缓存。

## 踩坑点

1. **大陆可达性波动**。jsDelivr 的国内节点受 ICP 影响，时好时坏。备好 `fastly` / `gcore` / `testingcf` 几个备用域名，前端做失败自动切换；重要业务别押注它。
2. **缓存刷新**。`@main` 引用有小时级缓存（官方口径约 12 小时），改图必须改名或主动 purge。实践上坚持"只增不改"最省心。
3. **仓库膨胀**。git 历史会永久保存每张图，务必只提交压缩后的 webp，原图留在本地冷备；单文件上限 50MB，实际建议控制在几百 KB。
4. **隐私与权限**。public 仓库全员可见，截图里的 token、内网地址先打码；PAT 用细粒度 token，只开 Contents 读写。
5. **API 限流与冲突**。批量上传控制并发，遇 403/429 退避重试；同名冲突时先 GET 拿最新 sha 再 PUT。

## 可复用建议

- 命名统一 `{yyyy}/{mm}/{hash8}.webp`，只增不改，URL 即内容寻址；
- 三级回退：`cdn.jsdelivr.net` → 镜像域名 → `raw.githubusercontent.com` 保底；
- 长期引用的图（如教程配图）打 release tag，用 `@v1.0.0` 引用，URL 永不漂移；
- 把上传逻辑封成 MCP 工具后，Agent 截图 → 压缩 → 上传 → 回链可以全程自动，写报告的体验顺滑很多。

## 总结

这套方案适合个人博客、社区帖、Agent 产出物这类"零成本、够用就好"的场景，不适合有 SLA 要求的生产业务。工程上记住三件事就够了：只增不改、压缩后再上传、多域名回退。剩下的工作，就是把它自动化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/d3260244bfb5ee8c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/b46e50a8a9759396.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/daca68ca5fa22b08.png)

