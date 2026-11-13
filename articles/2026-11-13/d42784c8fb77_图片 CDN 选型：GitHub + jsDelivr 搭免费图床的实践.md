---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 44287
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

写技术帖、文档，或者让 Agent 自动产出带图内容时，图片链接是绕不开的一环：截图和示意图需要一个稳定、可外链、成本可控的 URL。对 OpenClaw 这类自动化流水线来说，“Agent 生成 Markdown → 插入图片 → 发布”这条链路里，图片存储最容易被忽视，也最容易翻车。

## 问题

常见选项各有代价：

- **对象存储（OSS/COS）**：稳定，但要实名、可能绑域名、按量计费，个人项目维护成本高；
- **第三方免费图床**：有额度、限速和跑路风险，不适合长期外链；
- **GitHub raw**：免费且版本化，但 raw.githubusercontent.com 在国内访问不稳定，还有速率限制；
- **jsDelivr**：在 GitHub 仓库之上加一层全球 CDN，免费、免备案，国内可达性整体好于 raw。

结论：GitHub 仓库做存储源，jsDelivr 做分发，是个人和小团队性价比最高的组合——前提是接受它的缓存语义（见踩坑点）。

## 做法

1. **建一个纯图仓库**（如 `images`），按年份分目录，控制仓库体积；
2. **内容寻址命名**，天然不可变、避免同名覆盖：

```bash
sha=$(sha1sum "$f" | cut -c1-8)
name="$(date +%Y%m%d)-${sha}.webp"
```

3. **统一 URL 格式**：

```
https://cdn.jsdelivr.net/gh/<user>/images@main/<path>/<file>
```

4. **封装成脚本或 MCP 工具**：输入本地图片 → 压缩转 webp → 调 GitHub API（带 token）上传 → 返回 jsDelivr URL → Agent 拼进 Markdown。图床对上层流程是黑盒，后续换存储只改这一个工具。

## 踩坑点

- **jsDelivr 是强缓存**。同名路径覆盖旧文件后 CDN 短期不刷新，必须走 purge 接口：`curl -X POST "https://purge.jsdelivr.net/gh/<user>/images@main/<path>/<file>"`。purge 有频率限制，所以最省心的解法还是文件名带 hash，基本不需要 purge；
- **单文件上限 20MB**，超了直接 404。发布前统一压缩转 webp，顺便省流量；
- **`@main` 永远取最新**，牺牲缓存稳定性；要强不可变就打 tag，URL 写 `@v1.0`；
- **未认证 GitHub API 只有 60 次/小时**，自动化必须带 token，并做重试和本地暂存，失败不能丢文件；
- **jsDelivr 国内可用性会波动**。它本质是缓存层，仓库本身才是 source of truth；关键场景建议双写一份到 OSS 兜底；
- **别把仓库当垃圾场**，历史里塞满大二进制会让 clone 和 API 都变慢。

## 可复用建议

- 命名规范（日期 + hash）一开始就定死，后期迁移省一半力气；
- 上传逻辑收敛到一个工具/插件，Agent 只调接口、不感知存储细节；
- 加定时巡检脚本，抽样请求几个 URL 验证 200，CDN 异常时告警；
- 仓库即备份，CDN 挂了数据还在，心态会好很多。

## 总结

GitHub + jsDelivr 不是最稳的图床，但“免费、免备案、版本化、可自动化”四项全占。把它当成带缓存的分发层而不是存储层：命名不可变、上传收敛成工具、巡检做兜底，这条链路在个人自动化流水线里完全够用。等量级或稳定性要求上来，因为调用已经收敛，切换到对象存储的成本也很低。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/7e074f1275073f8a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/d61c569791d0e718.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/8b091b9ead9da2d7.png)

