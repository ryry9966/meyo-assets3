---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 44036
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景

写技术文档、跑自动化发布时经常撞上同一个问题：图放哪。Agent 生成的周报、插件的使用手册、自动化流程往博客 / 邮件 / RSS 里塞的截图，都需要一个稳定可外链的图片 URL。国内环境下这道选择题更麻烦。

## 问题：候选方案各有代价

- **对象存储（OSS/COS）**：稳定，但要备案域名 + 计费，个人用量的琐碎账单很烦
- **各类免费图床**：随时跑路，历史文章的图全挂
- **GitHub raw 链接**：`raw.githubusercontent.com` 国内直连基本不可用，部分场景 content-type 也不对

我的结论是组合方案：**GitHub 当存储层，jsDelivr 当 CDN 分发层**。jsDelivr 会镜像任意公开 GitHub 仓库，全球有节点，国内可用性常年够用（有波动，见踩坑）。

## 做法

1. 建一个公开仓库（比如 `images`），专门放图。
2. 外链格式：

```text
https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch>/path/to/img.png
```

用 `@main`，更推荐用 `@版本tag`——带版本号的路径缓存更持久，`@main` 内容变更后要靠 purge 刷新。

3. 上传自动化：写个小脚本（或包成 MCP 工具），本质就是调 GitHub Contents API：

```bash
curl -X PUT https://api.github.com/repos/USER/images/contents/2025/img.png \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -d '{"message":"add img","content":"'"$(base64 -w0 img.png)"'"}'
```

返回的 JSON 里有 commit 信息，顺手拼好 CDN URL 回传给 Agent 即可。

4. 缓存刷新：更新同名文件后主动 purge：

```bash
curl https://purge.jsdelivr.net/gh/USER/images@main/2025/img.png
```

更省心的做法是文件名带日期戳或 hash，永远新增不覆盖，直接绕开缓存问题。

## 踩坑点

- **单文件上限 20MB**：jsDelivr 不代理超限文件，直接 404。截图压缩后再传。
- **仓库别堆成垃圾场**：体积过大影响镜像效率，历史图片按年拆目录或归档到新分支。
- **`@main` 缓存不是实时的**：更新后可能十几分钟才刷掉，强迫症走 purge 接口或改用版本化路径。
- **合规灰区**：GitHub ToS 要求仓库以代码为主，纯图床属于擦边。控制规模、README 写清用途，别当业务级存储用。
- **文件名**：中文 / 空格 / emoji 的文件名 encode 后又长又易错，统一改成 `日期 + hash` 的 ASCII 命名。
- **国内可用性波动**：jsDelivr 时好时坏，重要发布页准备 raw 链接做降级。

## 可复用建议

- 把「传图 → 拼 URL → 返回链接」封装成一个 MCP 工具，Agent 写文 / 发帖时按需调用，体感接近原生图床
- 路径规范用 `/YYYY-MM/hash.png`，天然按时间归档
- 确定性要求高的场景（文档配图）用 git tag 锁版本：`images@v1.2/img.png`，永不失效
- 大图先压到 1MB 以内，PNG 转 WebP 能省一半以上带宽

## 总结

这套方案的本质是「拿免费存储换合规边界，拿 CDN 换可用性波动」。它不适合放业务关键资产，但对个人文档、Agent 产出物、自动化流程的配图完全够用：成本为零、链路全 API 化，非常适合塞进 Agent 工具链。缺点也想得清楚——别贪量，别放敏感图，重要内容永远留本地副本。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/7c45b814c2a28c92.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/3737f8bd3bdf8dae.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/c595857fc4a4cce0.png)

