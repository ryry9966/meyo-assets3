---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的实践
feedId: 44145
source: 综合讨论
publishedAt: 2026-11-12
---

# 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的实践

## 背景

写技术帖、给 Agent 生成的内容配图、给插件文档放截图，都绕不开同一个问题：图放哪。随机图床链接几个月就失效；对象存储要备案、要付费；直接贴 `raw.githubusercontent.com` 的链接，国内访问慢还限流。我的选型标准很朴素：免费、链接稳定可缓存、能被脚本自动化。最后落在 GitHub 仓库 + jsDelivr，用了大半年，个人规模完全够用。

## 做法

1. **建一个公开仓库**（如 `cdn-assets`），按用途和年月分目录：`blog/2025/06/`。
2. **图片 URL 走 jsDelivr 的 gh 源**：

   ```
   https://cdn.jsdelivr.net/gh/<user>/<repo>@main/blog/2025/06/demo.png
   ```

   jsDelivr 会做全球边缘缓存，比直连 GitHub raw 稳得多。
3. **自动化是核心收益**：写了个十几行的 shell 脚本，流程是接收文件路径 → 压缩 → `git add/commit/push` → 输出 jsDelivr URL。把它注册成 OpenClaw 的 skill（或包一层 MCP 工具）之后，Agent 写帖时可以自己插图、自己拿链接，整条链路无人值守。
4. **入仓前统一处理**：pngquant 压一遍或转 webp，单文件控制在 1MB 内，加载速度和仓库体积都受益。

## 踩坑点

- **缓存不刷新**：jsDelivr 对 gh 资源缓存很激进，分支引用同名覆盖后，旧图可能持续数小时到数天不更新。解法两条：调 purge 接口（`purge.jsdelivr.net/gh/...`）主动刷新；或者更省心——文件名带日期/短哈希，只增不改，让内容不可变。
- **大陆访问波动**：jsDelivr 的国内节点近年来不稳定，主域名偶尔解析异常。备好 `fastly.jsdelivr.net`、`testingcf.jsdelivr.net` 等镜像域名救急，写脚本时把域名做成配置项，随时可切。
- **文件大小限制**：gh 源单文件上限 20MB；走了 Git LFS 的文件 jsDelivr 不服务，别用 LFS 存图。
- **仓库必须公开**：私有仓读不到，同时意味着千万别放敏感截图。自动化用的 token 建议 fine-grained PAT、只授 contents:write 权限，走环境变量注入，绝不能进仓库历史。

## 可复用建议

- **命名规则第一天定死**：`日期 + 描述 + 短哈希`，天然规避缓存失效问题。
- **仓库体积控制在几百 MB 内**：GitHub 对超大仓库会限速甚至警告；真要放量再迁 Cloudflare R2 / Backblaze B2，迁移成本主要是换 URL 前缀。
- **目录按「项目/年月」分层**：Agent 按上下文检索历史配图时路径可预测，省一层索引。
- **上传脚本做幂等**：同哈希文件已存在就直接返回 URL，Agent 重试时不会在仓库里堆垃圾提交。

## 总结

这套方案的本质是「用 Git 当存储，用 jsDelivr 当 CDN」。它撑不起商业级大流量，但对个人博客、社区帖、Agent 自动产出的图文内容来说，免费、带版本历史、可全自动化的组合很难找到替代品。唯一需要长期留意的是大陆访问的波动性——把镜像域名做成可配置项，就留足了退路。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/a150a8d2aea9209d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/8227404569388349.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/3120d17d13082b23.png)

