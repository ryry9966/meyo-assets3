---
title: 从 Agent 产出到多平台发布：一套 Markdown 格式适配管线
feedId: 43944
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

用 Agent 写技术内容，产出几乎都是 Markdown。但"写完"离"发出去"还差得远：公众号编辑器只认内联样式的富文本，掘金、知乎各有 API 和语法限制，静态博客要 frontmatter，笔记工具还有自己的方言。早期的做法是每发一处手动调一遍格式，重复劳动，且经常翻车。在 OpenClaw 的工作流里，我把它做成了标准化管线，这里记录一下思路。

## 问题

拆开看是三个矛盾：

1. **方言差异**：CommonMark、GFM、平台私有语法互不兼容，表格、公式、Mermaid 的支持度参差不齐。
2. **AI 输出不稳定**：同一 prompt 下模型可能混用内联 HTML、自造标记、多余空行，格式不可复现。
3. **平台能力不对等**：公众号会静默剥离不认识的标签和外链图片，公式渲染规则各家独立。

## 做法

核心思路一句话：**规范输入 → AST 归一化 → 平台适配器**。

```
Agent 生成 → remark 解析 → lint 校验(失败回喂修复) → 统一渲染 → 适配器×N → 各平台
```

**第一步，约束输入。** 与 Agent 约定一个 Markdown 子集：只用 GFM，标题从 `##` 起，禁止内联 HTML，图片用相对路径占位。约定写进生成 prompt，但不依赖它兜底。

**第二步，lint 与修复回路。** 用 unified 生态解析成 mdast，跑自定义 lint：图片必须有 alt、表格不超宽、frontmatter 字段齐全。校验失败把报错回喂给 Agent 自修复，实测比在 prompt 里反复强调有效得多。

**第三步，统一中间表示。** mdast → hast 转成干净 HTML；代码块用 shiki 在构建期高亮；Mermaid 预渲染成 PNG；公式预渲染成 SVG；图片统一转存自有图床，顺带解决防盗链。

**第四步，平台适配器。** 每个目标平台一个小模块：静态博客直接输出 canonical Markdown + frontmatter；公众号用 serializer 把 class 内联成 style，标签降级为 section/p/span；掘金、知乎走 API 或生成可粘贴的 HTML。整条链封装成 CLI 和 MCP tool，Agent 一次调用完成"生成 → 校验 → 多平台产出"。

## 踩坑点

- 公众号会静默吞掉 `<details>`、部分表格样式，适配器要用它认的结构重写，别指望"先发出去再手动改"。
- 用正则改 Markdown 是经典事故源：竖线、嵌套列表、代码块里的伪语法都能让替换炸掉。一切转换基于 AST。
- SVG 嵌字体导出后会丢，Mermaid 图要么转 PNG，要么内嵌字体。
- frontmatter 里的冒号和多行文本要转义，模型特别爱在这里犯错。
- 发布要做幂等：用内容 hash 做版本键，重跑管线不会产生重复草稿。

## 可复用建议

1. 维护一张**平台能力矩阵**配置（是否支持表格 / 公式 / 高亮），适配器按矩阵自动降级。新增平台只需一行配置加一个 serializer。
2. lint 规则打成独立包，生成端和发布端共用同一套校验，避免两边标准漂移。
3. canonical Markdown 是唯一事实源，所有平台产物都是可重建的派生物——出问题永远回源修，不在下游打补丁。

## 总结

把 Markdown 当中间格式而非终点：输入端用 lint 约束 AI 输出，中间用 AST 归一化，输出端用适配器消化平台差异。管线跑通后，多平台发布从半小时的手工活变成一条命令，格式稳定、可回归、可重放。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/ed59bba19b65495c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/df03f9c1ea9117e5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/f2b8caf62d36ac31.png)

