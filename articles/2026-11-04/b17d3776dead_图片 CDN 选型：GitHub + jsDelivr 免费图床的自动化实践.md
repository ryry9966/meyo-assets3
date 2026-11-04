---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床的自动化实践
feedId: 43441
source: 综合讨论
publishedAt: 2026-11-04
---

## 背景

写社区帖子、博客，或者让 Agent 生成带图文档时，图片放哪是个绕不开的问题：本地路径别人看不见，商业图床要么收费要么限速，对象存储要绑卡还要走备案。对个人低频场景，我最终落在 GitHub 仓库 + jsDelivr 这条路上——仓库当存储层，jsDelivr 当分发层，零成本，且整条链路可以用脚本或 MCP 工具自动化。

## 问题

直接引用 raw.githubusercontent.com 有两个硬伤：部分地区访问慢甚至不稳定；没有 CDN 缓存，一张大图能拖垮整页加载。而上传环节如果靠手动，Agent 的产文流程就断了——需要一个可编程的「本地文件 → 公网 URL」通道。

## 做法

1. 建一个 public 仓库，如 `cdn-assets`，专库专用，不与代码混放。
2. 目录按 `img/YYYY/MM/` 组织，文件名用「日期-内容哈希」，如 `20250612-3f9a2b.png`。
3. 引用时走 jsDelivr 的 GitHub 源：

```text
https://cdn.jsdelivr.net/gh/<user>/<repo>@main/img/2025/06/20250612-3f9a2b.png
```

4. 上传自动化，最小实现是 Contents API：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/USER/cdn-assets/contents/img/2025/06/demo.png \
  -d '{"message":"upload","content":"<base64>","branch":"main"}'
```

5. 在此之上封装成一个小工具或 MCP tool：输入本地路径 → 读文件、算 md5 取前 8 位 → 拼路径 → 调 API → 返回现成的 `![](url)` 片段。Agent 写文时直接调用，产出的就是可发布文本。
6. 确需覆盖同名文件时，主动刷缓存：

```bash
curl https://purge.jsdelivr.net/gh/USER/cdn-assets@main/img/.../demo.png
```

## 踩坑点

- **缓存不刷新**：覆盖同名文件后，jsDelivr 大概率仍返回旧图。用哈希文件名规避——内容变、文件名变，天然幂等，Agent 重试也不会产生重复图。
- **可达性波动**：jsDelivr 的国内节点时好时坏。关键配图建议在正文同时留 raw 链接兜底，别把链路全押在一家 CDN 上。
- **定位与体量**：jsDelivr 官方定位是开源项目静态资源分发，单文件 20MB 上限。当私人相册高频滥用有被限制的风险；低频配图没问题，控制好频率和体积。
- **API 限流**：未认证 60 次/小时，务必带 PAT，且只授仓库写权限的最小 scope。
- **文件名**：中文、空格、大写都会在 URL 层面出幺蛾子，一律小写加连字符。

## 可复用建议

- 把上传逻辑做成独立 MCP 工具（约 60 行代码），任何 Agent 会话里都能「发图即得链接」——这套模式比图床本身更值。
- 同仓库维护一份 `index.json`，记录原始文件与 CDN URL 的映射，将来迁移到对象存储有据可查。
- 命名即版本：哈希文件名让上传变成幂等操作，可以放心重试和并发。

## 总结

GitHub + jsDelivr 适合个人博客、社区帖子、Agent 生成文档这类低频配图场景：零成本、全链路可自动化、无供应商锁定。代价是可达性依赖第三方、内容必须公开。把它当工程里的「够用方案」而非终态，配合哈希命名和上传工具化，体验已经相当顺滑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/b4f38f1b14305a7f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/925e14d84b1ff0d8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/a50db5ddf2a237ef.png)

