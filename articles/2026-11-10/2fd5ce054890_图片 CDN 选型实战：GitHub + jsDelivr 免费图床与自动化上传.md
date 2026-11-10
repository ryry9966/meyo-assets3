---
title: 图片 CDN 选型实战：GitHub + jsDelivr 免费图床与自动化上传
feedId: 43989
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景

写技术文档、跑 Agent 自动化时经常要产出截图：MCP 工具返回的运行快照、插件生成的报表图、博客配图。这些图片需要一个稳定的公网 URL，否则链接要么失效，要么加载很慢。对象存储要付费和备案，商业图床随时可能关停，相比之下 GitHub 仓库 + jsDelivr 的组合几乎是零成本方案：仓库当存储层，jsDelivr 当全球分发层，两边都不需要额外注册。

## 问题

核心诉求有四条：链接长期稳定、国内可访问（或至少可控）、支持 API 自动上传、成本为零。另外在我们的场景里，上传动作往往发生在 Agent 流程中间——插件截完图后要立刻把 URL 塞进生成的 Markdown 里，所以必须能封装成一个可被工具调用的原子操作。

## 做法

1. 建一个公开仓库，比如 `img-bed`，纯放图片，不混代码。
2. 申请 Fine-grained PAT，只授予该仓库的 Contents 读写权限，不要用全量 token。
3. 上传走 GitHub Contents API，请求体带 base64 编码的图片：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/USER/img-bed/contents/2025-06/abc123.png \
  -d '{"message":"add img","content":"'"$(base64 -w0 img.png)"'"}'
```

4. 拼接 jsDelivr 链接：`https://cdn.jsdelivr.net/gh/USER/img-bed@main/2025-06/abc123.png`。
5. 把 3、4 两步封成一个小函数，注册成 MCP tool：输入本地路径或 base64，输出 URL 字符串，Agent 直接调用。

## 踩坑点

- **缓存不可控**：jsDelivr 对同名文件缓存很激进，覆盖上传后 CDN 上还是旧图。解法是文件名带内容哈希（前 8 位足够），改图等于新文件名，天然刷新。
- **国内连通性**：jsDelivr 在大陆的可用性时有波动。重要场景备一条 `raw.githubusercontent.com` 回退链接，或套一层自己的 Cloudflare Worker 反代。
- **仓库体积**：GitHub 对仓库有软性建议上限，图片堆多了 push 会变慢。按月份分目录，超过几 GB 就换新仓库。
- **内容合规**：公开仓库等于公开图片，带敏感信息的截图（内网地址、token）绝不能传；工具里最好加一层尺寸/类型白名单或人工确认。
- **API 限额**：认证后 5000 次/小时，常规自动化没问题，但别在循环里无脑重试。

## 可复用建议

- 文件命名统一 `{内容哈希前8位}.{ext}`，放进 `yyyy-mm/` 目录，去重和缓存失效都靠它解决。
- 上传函数返回 JSON，同时给出 jsDelivr 和 raw 两个 URL，调用方自己选。
- 上传前本地压一遍图（pngquant 或 sharp），单张控制在 1MB 内，加载速度和仓库体积都受益。
- token 放环境变量，别写进 agent 配置文件，日志里注意别打印请求头。
- 迁移时 URL 与仓库路径一一对应，写个脚本即可批量重建，不会被方案锁死。

## 总结

这套方案的定位很清楚：个人博客、文档、Agent 产出物这类中低流量场景下的免费、稳定、可全自动化的图床；不适合对外提供大规模热链，也不适合放敏感内容。工程上的关键是把「上传 + 拼 URL」封装成可复用工具，文件名带哈希解决缓存问题，同时留好回退和迁移路径。一小时搭好，长期够用，投入产出比是值得的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/3b061a36efeef2ae.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/8d0812a34213ec8d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/8fdd30d32d1eaa13.png)

