---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 44143
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景

社区里发帖、写文档、给 Agent 输出配可视化卡片，都绕不开贴图。截图放本地没人看得见，传第三方图床又常遇到限速、外链失效、服务关停。作为长期跑自动化的用户，我需要一条"脚本可调、链接稳定、零成本"的图片链路，最后落在 GitHub 仓库 + jsDelivr 这套组合上。用了几个月，把实践和坑记录一下。

## 问题

选型时有几个硬指标：

- **可编程**：Agent/MCP 工具能直接 curl 或走 API 完成上传，不依赖网页手动操作
- **外链稳定**：图片有正式 CDN 直链，可直接嵌 Markdown
- **版本可追溯**：图片本身进 Git，删错能找回
- **免费且限额宽松**：个人文档量级够用

SM.MS 这类免费床限额和存活时间不可控；对象存储 + CDN 要绑域名、备案、付费。GitHub + jsDelivr 恰好补位。

## 做法

1. 建一个公开仓库（如 `imgs`），专放静态资源
2. 申请 fine-grained PAT，只授权这一个仓库的 Contents 读写，权限最小化
3. 上传走 Contents API，不需要本地 git：

```bash
curl -X PUT https://api.github.com/repos/USER/imgs/contents/2025/06/demo.webp \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"message":"add img","content":"BASE64"}'
```

4. 拼 jsDelivr 链接即可外链：
   `https://cdn.jsdelivr.net/gh/USER/imgs@main/2025/06/demo.webp`
5. 封装成脚本后挂到 OpenClaw 的 MCP 工具里：Agent 截图 → 压缩成 webp → 上传 → 返回 CDN 链接，全程无人工。

## 踩坑点

- **缓存不刷新**：同名覆盖后 jsDelivr 会继续返回旧缓存。用"日期 + 内容哈希"命名，或改用 `@版本标签`；急用可调 `purge.jsdelivr.net` 强制刷新。
- **大陆访问波动**：`cdn` 主域名在大陆时好时坏。配置里备好 fastly / gcore / testingcf 几个镜像域名做回退，发文前用脚本探活再落链接。
- **仓库体积**：单文件超 50MB 告警，超 100MB 直接被拒。截图先压 webp（质量 80 通常省 60%+），仓库控制在 1GB 内比较安全。
- **滥用风险**：拿它分发大文件、做视频床，可能被 jsDelivr 拉黑整个仓库。图床归图床，别越界。
- **隐私**：公开仓库 = 图片公开。带敏感信息的截图务必脱敏后再上传。

## 可复用建议

- 命名规范用 `{日期}/{内容sha1前8位}.webp`，天然去重，也天然避开缓存坑
- 上传脚本加并发队列 + 失败重试；认证后限额 5000 次/小时，个人批量截图完全够
- 保留原始 raw.githubusercontent 链接作为备份字段，jsDelivr 万一失效还能换域名拼回去
- 把"压缩 → 上传 → 返回链接"做成一个 MCP tool，比每次让 Agent 现场拼命令靠谱得多

## 总结

这套方案本质是"把 Git 当对象存储，把 jsDelivr 当 CDN"。它没有 SLA，不适合承载关键业务资产；但对个人文档、社区发帖、Agent 自动化产物这些场景，成本为零、链路全脚本化、图片有版本历史，性价比很难被拒绝。真正的工程量不在搭建，而在命名规范、缓存策略、回退域名这三件小事上——把这三件事做对，它就能安静地跑很久。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/bd256646363dcbbc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/1f2ad1f8aecc3b3b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/cf0b5a0bd18ec244.png)

