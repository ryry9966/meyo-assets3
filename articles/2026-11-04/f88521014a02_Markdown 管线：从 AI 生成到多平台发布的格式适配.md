---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 43384
source: 综合讨论
publishedAt: 2026-11-04
---

## 背景

最近半年，我的内容生产入口基本换成了 agent：让 OpenClaw 挂上 MCP 工具做调研、出初稿，我再改。但真正的痛点不在生成，而在发布——同一篇内容要进静态博客、GitHub README、公众号、知乎，各平台的 Markdown 方言差异很大，手工逐个排版的时间远超写作本身。

我的方案：把 Markdown 当作整条管线的 canonical 中间表示（IR），生成端只产出受限子集，发布端为每个平台写一个 adapter。

## 问题

实际遇到的不兼容点：

- 公众号不认原生 Markdown，代码块无高亮，图片必须走素材库外链；
- 知乎会把 frontmatter 直接渲染成正文，脚注支持不稳定；
- H1 层级、硬换行、`<br>` 的行为各家 renderer 不一致；
- agent 输出不可控：会塞 HTML 标签、把代码块嵌进引用，偶尔输出整段多余围栏。

## 做法

管线三步，用 remark（mdast）实现，不到两百行：

1. **约束生成端**。在 agent 的 system prompt 里声明只允许 CommonMark 子集：h2/h3、列表、代码块、表格、引用，禁止内联 HTML。只靠 prompt 不够，再写一个 schema-lint 作为 MCP 工具暴露给 agent，输出前自检，不合规直接 reject 让它重写。
2. **归一化**。把原始 Markdown parse 成 mdast 做确定性变换：标题统一 h2 起步、frontmatter 摘成元数据对象、图片链接改写为本地路径（发布时再按平台上传替换）、剥掉 HTML 注释。
3. **平台 adapter**。每个 target 一个 renderer：静态站直出 + 注入 frontmatter；公众号走 mdast → 内联样式 HTML，代码块预渲染成带颜色的 span，图片先传素材库换链接；知乎对复杂表格降级为列表。全部先 dry-run 输出到本地预览目录，人工确认后再推。

## 踩坑

- **正则改格式必翻车**。早期用正则删 H1、改图片链接，一遇嵌套代码块就崩。切到 AST 后这类问题清零：遍历节点改属性，永远比文本替换可靠。
- **agent 不守约定是常态**。lint 只报错不够，要给出行号和违规原因，retry 一次成功率才能上九成。剩下的问题里，某些模型反复在引用块里包代码块，我干脆在 normalize 加了一条规则：`blockquote > code` 自动提升为顶层代码块，比反复 retry 便宜。
- **图片是最大的平台耦合点**。公众号防盗链 + 素材库接口限制，Base64 内联又超体积限制，只能在 adapter 层做“先上传、再替换链接”，并把上传结果做映射缓存，避免重复传。
- **硬换行**。公众号把单换行渲染成换行，GitHub 渲染成空格。归一化阶段统一把单换行合并为段落，需要强换行处显式处理。

## 可复用建议

- 用 AST 而非正则做格式转换，remark/hast 生态够用；
- 把 lint 做成 agent 可调用的工具，让模型输出前自查，比事后清洗便宜；
- canonical 版本单独存一份并带 hash，多平台产物由它派生，diff 时能定位到源头；
- 每个 adapter 配一个最小样例，CI 里跑渲染 smoke test，平台规则一变就能发现。

## 总结

这条管线本身不复杂，价值在于把“格式适配”从每次手工排版变成一次性的工程投入：来源受限、AST 归一、平台 adapter、渲染验证，四段各管各的。之后接新平台只需新增一个 adapter 文件，上游完全不用动。对做内容自动化的同学，建议先从 lint 工具做起——投入最小、收益最快的一环。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/76a42c3446f1ee1c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/3e3b82b7239796cd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/38cecee5f7232149.png)

