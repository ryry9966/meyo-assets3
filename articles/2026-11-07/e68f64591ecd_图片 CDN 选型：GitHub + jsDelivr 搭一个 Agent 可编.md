---
title: 图片 CDN 选型：GitHub + jsDelivr 搭一个 Agent 可编程调用的免费图床
feedId: 43680
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

在 OpenClaw 的日常使用里，图片产出的场景比想象中多：Agent 生成知识卡、插件文档要配图、自动化流水线要落截图、博客和 README 要引用素材。这些图都需要一个"链接稳定、可被程序拼接、长期有效"的托管出口。

## 问题

我先后试过几条路：

- **SM.MS、imgur 这类免费图床**：有防盗链与限流，部分域名在部分网络环境下不可达，且链接格式不可预测，Agent 没法可靠拼接；
- **GitHub raw 链接**（raw.githubusercontent.com）：链接稳定可拼接，但国内直连经常超时；
- **自建对象存储**：稳定，但要挂域名、备案、计费，对小体量素材过重。

折中方案：仓库仍放 GitHub（保住版本管理与 raw 兜底），分发走 jsDelivr 的 gh CDN。

## 做法

1. 建一个 public 仓库，比如 `assets`，按 `{yyyy-mm}/{project}/` 分目录；
2. URL 规则固定为：`https://cdn.jsdelivr.net/gh/<user>/<repo>@main/<path>/<file>.png`。用户名、仓库名、路径规则确定后，Agent 可以直接拼 URL，不需要任何 API 查询；
3. 把上传动作封装成一个 CLI / MCP tool：接收本地图片 → 按内容 hash 重命名 → git commit & push → 返回拼接好的 jsDelivr URL。在 OpenClaw 里注册成工具后，对话里一句"把这张图发到图床"就能拿到最终链接；
4. 文件名带内容 hash（如 `a3f2c1.png`），天然去重，也规避缓存问题。

## 踩坑点

- **缓存不可覆盖**：jsDelivr 对文件有缓存，同名覆盖后线上可能仍是旧图。解法是 append-only：文件只新增不修改，命名带 hash；
- **单文件体积**：超过约 20MB 的文件不予分发，截图先压缩（转 webp 通常能砍掉 60% 以上体积）；
- **国内可达性会波动**：jsDelivr 历史上经历过解析异常，不能当唯一出口。我的做法是工具同时返回 raw 链接和 CDN 链接，按环境降级；
- **public 才能分发**：私有仓库不行，所以不要把带敏感信息的截图推进去，工具里加一步文件名校验或脱敏提醒；
- **提交频率**：批量插图时合并成一次 commit，避免触发风控。

## 可复用建议

- 命名规范写进工具而不是靠自觉：`{yyyy-mm}/{project}/{8位hash}.{ext}`；
- 给工具加一个 `--check` 子命令，HEAD 请求批量校验历史链接，404 及早发现（比如仓库改名会全量失效）；
- 如果素材是 Agent 长期产出，考虑按月打 tag，用 `@tag` 引用获得不可变链接；
- 量级上去之后（几个 GB 以上或要求强可用性），该迁对象存储就迁。这套方案的价值边界在"个人与文档级"。

## 总结

GitHub + jsDelivr 不是生产级 CDN 的替代品，但它把"链接可拼接、版本可追溯、零成本"三件事同时做到了，非常契合 Agent 自动化对图片链接的最低要求：**给定输入，URL 确定**。把它封装成几十行的 MCP tool 之后，插图这件事就从手工操作变成了对话中的一句话。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/28e955bd9dc354a5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/de026f49fb177c02.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/62926bc69215c9da.png)

