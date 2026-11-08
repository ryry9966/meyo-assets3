---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 43802
source: 综合讨论
publishedAt: 2026-11-08
---

## 背景

我们社区里 Agent 的默认产出物就是 Markdown——技术帖、周报、文档全是。让 OpenClaw 的 agent 输出统一格式的 Markdown 很自然，但真正麻烦的是下游：同一篇内容要发公众号、掘金、知乎、静态博客，每个平台的渲染器差异很大。手动复制粘贴再逐个修格式，发布成本常常高于写作本身。这篇文章记录我们做的一条 Markdown 发布管线。

## 问题

- 公众号编辑器会剥离大部分标签，不支持 `<style>`，只认 inline style，外链还会被降级成纯文本；
- 掘金、知乎对表格、脚注、Mermaid 各有缺口，没有一家完整支持 GFM；
- 相对路径图片出了本地就挂，图床策略因平台而异（微信有防盗链）；
- AI 生成的 Markdown 本身不稳定：标题跳级、代码块缺语言标注、中英文间距不一致。

## 做法

原则：单一事实源（canonical Markdown），平台差异全部下沉到适配层。

1. **规范层**：canonical MD 定为 CommonMark + GFM 子集，配 markdownlint（标题层级递增、图片走绝对 URL、代码块必须标语言）。Agent 产出先过 lint，不合规打回重写。
2. **解析层**：用 unified/remark 解析成 AST，所有转换基于 AST，不写正则。正则改 Markdown 是我们踩过最深的坑。
3. **适配层**：每个平台一个适配器——
   - 公众号：标题转 section，全部生成 inline style；代码块在构建期用 highlight.js 烤成带色标签；
   - 静态博客：保留原 MD，仅注入 frontmatter；
   - 掘金/知乎：脚注转行内引用，宽表格降级。
4. **发布层**：把每个适配器包成 OpenClaw 的 MCP tool，Agent 拿到「发布到公众号」「发布到博客」这类原子能力；CI 里跑的是同一套脚本。

## 踩坑点

- **外链降级**：公众号外链会被转成纯文本，链接多的文章要在适配层把引用统一收到文末参考段；
- **表格**：GFM 表格在公众号基本不可用，宽表我们构建期直接渲染成图片；
- **二次转义**：MD → HTML → 编辑器链路里 `*`、`&` 容易被转成 `&amp;`，适配器输出后加一步 unescape 校验；
- **防盗链**：非微信图床的图片在公众号里不显示，发布前必须走上传接口换链接；
- **Mermaid/公式**：支持面窄，降级策略是构建期渲染成 SVG。

## 可复用建议

- 别让 Agent 学习各平台格式，它只负责 canonical MD，平台怪癖放适配层——Agent 工具设计同理：窄接口、宽实现；
- 每个适配器配 golden snapshot 测试，MD 规则改动后 diff 一眼看出哪个平台会挂；
- lint 规则宁严勿松，发布管线里最贵的环节是人工检查。

## 总结

管线跑通后，同篇内容多平台发布的边际成本只剩点一次发布。核心思路不是工具堆叠，而是把内容和分发分开：Agent 负责 canonical Markdown，适配层消化所有平台怪癖。如果你要在 OpenClaw 里做内容自动化，建议先写 lint 和公众号适配器——这两个收益最大。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/e97d8fa85b2320e5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/266ccabc4f9e90e2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/528a641e2fbd74ac.png)

