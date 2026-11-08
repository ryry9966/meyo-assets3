---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭一个免费且可自动化的图床
feedId: 43823
source: 综合讨论
publishedAt: 2026-11-08
---

## 背景

做 Agent 工作流、写博客、让机器人发报告时经常要贴图。图放本地没法分享，放 OSS 要绑域名、按流量付费，各种免费图床说倒就倒。最后我选了 GitHub 仓库 + jsDelivr 的组合：存储交给 GitHub，加速交给公共 CDN，上传走 API 全自动化，成本为零。用了大半年，记录一下做法和坑。

## 做法

1. **建一个公开仓库**，比如 `images`，按 `2024/06/xxx.png` 分目录。
2. **生成 Fine-grained PAT**，只授权这一个仓库的 Contents 读写权限，不要用全仓库权限的 classic token。
3. **用 Contents API 上传**。一次 PUT 就够：`PUT /repos/{user}/images/contents/{path}`，body 里放 base64 内容和 commit message。十行脚本就能封装成 `upload(file) -> url`。
4. **引用地址走 jsDelivr**：`https://cdn.jsdelivr.net/gh/{user}/images@main/2024/06/xxx.png`。全球 CDN 分发，Content-Type 正确，Markdown 里直接贴。

如果你在用 OpenClaw，建议把第三步封装成一个 MCP tool：Agent 需要插图时自己调用，拿到 URL 直接写进回复或文档，全程无需人工干预。这是这套方案自动化价值最大的一步。

## 踩坑点

- **不要覆盖同名文件**。jsDelivr 对 GitHub 内容缓存很久，同名覆盖后 CDN 上还是旧图，刷新基本指望不上。解法：文件名带时间戳或内容 hash，让每个 URL 天然不可变。
- **国内可用性会波动**。jsDelivr 在大陆的解析策略调整过几次，部分地区、部分运营商会抽风。建议主链路用 jsDelivr，脚本里保留 `raw.githubusercontent.com` 兜底；对可用性零容忍的业务别把宝押在免费 CDN 上。
- **注意体积与合规**。单文件控制在 20MB 内（jsDelivr 对 gh 源有上限）；仓库是公开的，截图里的 token、内网地址先打码；别拿它当网盘囤大文件，容易触发滥用限制。
- **Token 管好**。自动化脚本走环境变量或 secret，不硬编码进仓库，定期轮换。

## 可复用建议

- 文件名规则统一成 `{日期}/{sha256 前 12 位}.{ext}`，幂等且天然去重。
- 上传函数里做限速和重试，批量导图时别一口气打满 API 配额。
- 工具返回双 URL（jsDelivr + raw），下游按场景自选。
- 量级上来后，平滑迁移路径是对象存储 + 自定义域名，仓库目录结构可原样搬走。

## 总结

这套方案的本质是：**用 Git 做存储层**（免费、有 API、可版本化），**用公共 CDN 做分发层**（免费、快、但不可控）。它适合个人博客、文档、Agent 产出插图这类低频写入、只读分发的场景；不适合当网盘，也不适合生产级强可用需求。把上传封装成 MCP 工具之后，Agent 就拥有了开箱即用的贴图能力——对 OpenClaw 用户来说，这才是选它的核心理由。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/7cb753420d310d5d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/d77b80b0d97b90ff.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/be8530ba61f0b2f0.png)

