---
title: Markdown 管线实践：从 Agent 生成到多平台发布的格式适配
feedId: 43523
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

在 OpenClaw 的典型用法里，Agent 的产出物几乎都是 Markdown：调研纪要、技术帖、周报、文档 PR 描述。但"写完"只是开始——同一篇稿子要发公众号、知乎、掘金、自建博客，各家对语法的支持参差不齐。手动复制粘贴每次都要修一遍格式，量一大就不可维护了。

## 问题

核心矛盾在于：AI 生成的是一份"理想化"的 GFM，而发布端各自裁剪。

- 公众号编辑器只认带内联样式的 HTML，标准 Markdown 粘进去，表格和代码高亮全丢；
- 知乎会吞掉部分 HTML 标签和脚注；
- 静态博客需要 frontmatter，但直接贴到社区平台会被当成正文渲染出来；
- 图片最麻烦：外链有防盗链，平台图床互不相通。

## 做法

我的管线分四层，全部基于 unified/remark 的 AST，不碰正则：

1. **规范化（normalize）**：唯一事实源是 Git 里的一份 canonical Markdown（GFM + 自定义 frontmatter）。先用 remark-lint 统一标题层级、列表缩进、图片引用格式，保证后续所有 transform 面对同构输入。

2. **AST 变换（transform）**：每个目标平台写一个 remark 插件。公众号适配器遍历 mdast，把 heading、code block、table 编译成带内联样式的 HTML 节点；博客适配器则提取 frontmatter 生成 Hugo 兼容格式。关键原则：**只在 AST 层面操作**，代码块内部内容永远当字符串原样保留。

3. **资产处理（assets）**：发布前把远端图片下载，按平台重新上传（公众号走素材接口，博客进本地仓库），再把新地址写回 mdast，避免源图床挂了导致整篇图裂。

4. **编排（orchestrate）**：前三步封成 CLI，再包成 MCP tool。Agent 侧只需调用 `publish(article_id, targets=["wechat","blog"])`，管线返回每个平台的落地链接和告警列表。

## 踩坑点

- 最早用正则做公众号适配，被嵌套代码块里的 `###` 注释搞得全军覆没，这是转向 AST 的直接原因；
- 公众号会剥掉 `<style>` 和 class，所有样式必须内联到每个标签上，一个都不能省；
- 脚注和数学公式不是处处支持，现在的策略是显式降级：适配器遇到不支持的节点就记 warning，绝不允许上线后悄悄丢内容；
- 任务列表（`- [ ]`）在多数平台渲染成奇怪的字面量，统一降级为普通列表；
- 图片重传后记得清理本地缓存，否则构建目录会指数膨胀。

## 可复用建议

- canonical 源文件进 Git，发布产物不进，diff 永远针对源做；
- 每个 transform 准备 3~5 个 fixture 文档做快照测试，改插件后跑一遍，立刻知道有没有弄坏其他平台；
- 降级策略写进适配器代码，不要靠人肉记忆；
- frontmatter 的 key 保持平台无关（title/date/tags/summary），平台差异全部放到适配器里补。

## 总结

这条管线本质上是把"格式适配"从一次性手工劳动变成可测试的代码。AST 是分水岭：正则方案在第一个嵌套代码块就会崩。对 OpenClaw 用户来说，把管线封装成 MCP tool 之后，Agent 就能"生成—校验—发布"一条龙，人只在告警出现时介入。花一个周末搭好骨架，之后每接入一个新平台，只是新增一个适配器而已。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/2f2556941bcdabb8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/51fd2282b5806492.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/18efdc30753da8aa.png)

