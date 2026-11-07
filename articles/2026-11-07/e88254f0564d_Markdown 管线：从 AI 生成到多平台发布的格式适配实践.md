---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配实践
feedId: 43736
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

LLM 默认输出 Markdown，这让它成了 Agent 产出内容的事实中间格式。但发布目标从来不止一个：Hugo 静态站、GitHub、公众号、知乎/掘金，各自的格式方言差别很大。早期我们的做法是让模型直接"转成公众号格式"，结果每次输出都不一样，还经常丢内容。后来把这件事当成一条工程管线来做，才稳定下来。

## 问题

Markdown 看似标准，实则方言丛生：frontmatter、脚注、GFM 表格、代码围栏、图片路径、内联 HTML。目标端限制又各不相同：公众号只认内联样式的 HTML，图片必须传到自家 CDN；知乎不支持脚注；静态站要求严格的 frontmatter 和标题层级。核心矛盾是：**AI 生成是概率性的，而格式适配需要确定性**。

## 做法

1. **定义 canonical markdown 作为单一事实源**，约束写进 lint：全文只有一个 H1、元信息进 frontmatter、图片用稳定文件名引用、禁用内联 HTML、表格保持简单。
2. **AI 输出先过 normalize**：修标题层级、清理围栏语言标签、剔除模型偶尔的寒暄句。注意这一步也用代码，不用 prompt。
3. **用 remark 解析成 AST**，所有适配器都是 mdast 到目标格式的纯函数：
   - hugo：补全 frontmatter、转 shortcode；
   - wechat：渲染为内联 CSS 的 HTML，图片先上传再改写 URL；
   - zhihu/juejin：脚注降级为行内引用，剥离不支持的语法。
4. **CI 跑 dry-run**：每个适配器输出做快照测试，加死链与图片路径检查。
5. **适配器包成 MCP 工具暴露给 Agent**，但工具只做确定性转换，Agent 负责编排（选目标、填元数据、触发发布），真实发布前强制 dry_run 加人工确认。

```text
pipeline/
  lint.mjs        # canonical 规则检查
  normalize.mjs
  adapters/
    hugo.mjs
    wechat.mjs
  snapshot/       # 各适配器输出快照
```

## 踩坑点

- 最初用正则改 Markdown，嵌套代码块一出现就崩，换 AST 后才稳。
- 让 LLM 直接做格式转换是大坑：它会顺手"润色"，丢段落、改数字，且不可复现。转换必须写死在代码里。
- 中文两侧的下划线常被误判成斜体，中文内容的强调语法要单独处理。
- 公众号图片外链会被防盗链拦截，必须上传后改写；知乎粘贴 HTML 会吞样式，要贴着它的标签白名单来。
- 模型偶尔在正文里再写一个 H1，发到静态站直接破坏页面结构——lint 加一条规则即可拦住。
- 别在适配器里内联高亮库的 class，多数平台会用自己的高亮器重新渲染。

## 可复用建议

- canonical 格式写成一份短文档，和 lint 规则放同一仓库，改格式必须连规则一起改。
- 每个适配器配两三个固定样例的快照测试，改转换逻辑先看 diff。
- 原始 AI 输出和人工编辑后的 canonical 分开存，方便回溯"是哪一层改坏的"。
- 适配器保持无状态纯函数，重试、上传、发布等编排逻辑放外层。

## 总结

把 Markdown 当成 IR（中间表示）而不是"通用格式"，是这条管线里最值钱的认知转变。AI 负责生成，代码负责转换，Agent 负责编排，三者边界划清之后，多平台发布就从碰运气变成了普通的 CI 任务。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/021cf8bf0259ae1f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/a147ab371135ea36.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/eca7eb0fcd8a3bfd.png)

