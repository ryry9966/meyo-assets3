---
title: Markdown 管线实践：AI 生成内容的多平台发布适配
feedId: 43529
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

用 OpenClaw 的 agent 产出技术文章是社区的常见用法：搜资料、写初稿、配示例代码，一条链路跑完。但初稿落地时问题才出现——AI 输出的是标准 Markdown，而发布端各说各话：公众号编辑器只认内联样式的 HTML，知乎对脚注和任务列表支持参差，静态博客又要 frontmatter 和固定图片路径。过去我的流程是复制、粘贴、手动调格式，一篇稿子在发布端的耗时比写作还长。这篇文章记录我后来搭的这条 Markdown 管线。

## 问题

拆开看是四类：

1. 平台语法差异：同一份 Markdown，五个平台渲染出五种结果；
2. AI 输出不干净：幻觉图片链接、mermaid 块、正文里重复的 H1、列表缩进忽 2 空格忽 4；
3. 资源处理：远程图要转到各平台自己的存储，代码高亮要预烘焙；
4. 重复劳动：每加一个发布平台，手工适配成本线性增长。

## 做法

管线分五层，前四层与平台无关：

**1. 规范源。** 只维护一份 canonical Markdown。元数据全放 frontmatter，图片统一用相对路径引用本地文件，正文禁用内联 HTML。写法用 markdownlint 固化，agent 生成时就按规则约束，比事后修便宜。

**2. 清洗层。** 用 remark 把 Markdown 解析成 mdast，在 AST 上清洗：对 AI 给出的图片链接发 HEAD 请求探测，404 的直接删；mermaid 块要么转静态图要么去掉；把文档第一个 H1 提出来写回 frontmatter 的 title。

**3. 平台 profile。** 每个目标平台一个 transformer，签名统一为 `mdast -> 平台格式` 的纯函数。公众号的 transformer 输出内联样式 HTML，代码块用 shiki 提前染好色；知乎走富文本接口，脚注转文内引用；静态站几乎透传，只补全 frontmatter。

**4. 图片管线。** 上传封装成独立的 MCP tool：输入本地路径，返回平台 URL。transformer 里只做 URL 重写，其他环节永远只见本地路径。

**5. 预览与发布。** 渲染各平台结果生成 diff 预览，人工确认后逐个调用发布接口。整体封装成一个 OpenClaw skill，一条命令跑完全程。

## 踩坑点

- **正则替换必炸。** 第一版用正则处理嵌套列表，代码块里的 `#` 注释被当成标题，解析错位。切到 mdast 后再没出过这类问题。
- **公众号剥属性。** `class` 全部无效，样式必须内联到 `style`，且转换要幂等——否则重复发布一次，样式叠加一层。
- **幻觉链接要探测。** AI 生成的图片 URL 看着很合理，直接上传必 404，发布前统一探测一次。
- **frontmatter 会漏。** 知乎把 `---` 渲染成分隔线，元数据必须在 transform 里剥干净。
- **表格宽度。** AI 爱写宽表格，移动端直接撑破，清洗层加一条规则：超宽表格转列表。

## 可复用建议

- transformer 保持纯函数，用三五篇固定样例文章做快照测试，改平台逻辑不怕回归。
- 每个平台 profile 附一份"已知限制"清单，发布前让 agent 自检一遍。
- 图片重写放在最后一步，前面所有层只操作本地路径。
- 新平台先写渲染预览、再接发布接口，确认渲染对了才碰 API。

## 总结

管线化之后，新增一个发布平台大约半天写完 transformer；从 AI 产出到多平台发布，从过去近一小时的手工适配降到几分钟的预览确认。核心思路三句话：canonical Markdown 是契约，AST 变换代替文本替换，平台差异和资源处理全部后置到发布层。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/104dca0b7ddfe631.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/c5d75c0f4e9c2776.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/13cf23efc61be1d9.png)

