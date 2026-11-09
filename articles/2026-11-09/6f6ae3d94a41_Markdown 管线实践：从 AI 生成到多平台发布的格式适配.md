---
title: Markdown 管线实践：从 AI 生成到多平台发布的格式适配
feedId: 43904
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

在 OpenClaw 的工作流里，Agent 承担了大量的初稿生产：技术帖、release note、调研整理，产出统一是 Markdown。但真正发布时，公众号、知乎、掘金、静态站各自的渲染规则差异很大，把同一份 md 直接丢给多平台，几乎必出问题。我们把这条链路抽成了一条适配管线，跑了半年，这篇记录它的结构和踩坑。

## 问题

三个核心矛盾：

1. **AI 产出不稳定**：同类任务，模型可能给你内联 HTML、mermaid 块、错误层级的标题、`[^1]` 脚注，甚至把整段正文包进 ` ```markdown ` 围栏里。
2. **平台限制差异大**：公众号不支持外链和脚注语法、图片必须传自家 CDN；表格在移动端极易溢出；各平台对数学记法的支持也不一致。
3. **人写和 Agent 写走两条路**：Agent 直接吐"平台定制版"，人工再改一遍，最后没人说得清哪份是 source of truth。

## 做法

管线拆成四段，全部基于 unified/remark 生态，操作对象是 mdast：

1. **规范化**：任何输入先转成 canonical markdown。固定规则：抽掉首个 H1 写入 frontmatter 的 title、剥内联 HTML、补齐代码围栏语言、图片相对路径解析为绝对 URL。用 remark-stringify 重新序列化，保证幂等——同一输入跑两次结果 byte 级一致。
2. **元数据层**：title/tags/date/summary 进 frontmatter，正文只承载内容，下游渲染器不再猜标题。
3. **平台适配器**：每个平台一个纯函数，canonical md 进、平台格式出。公众号适配器做三件事：外链转文末引用（`[1] 标题 URL`）、标题转加粗段落或内联样式 HTML、mermaid 块用 mermaid-cli 离线渲染成 SVG 后替换，图片走素材接口换链。掘金/知乎适配器基本只做 frontmatter 剥离和数学记法转换。
4. **校验与编排**：发布前跑链接可达性、空 alt 图片、围栏闭合检查。管线同时封装成 CLI 和 MCP tool，Agent 调 `publish`，人工跑同一条命令，保证同源。

## 踩坑点

- 嵌套围栏（` ```markdown ` 包正文）会让 mdast 解析直接断掉。解法：解析前检测 fence 长度，动态加长外层围栏再喂进去。
- 尾随双空格换行会被部分处理器吞掉，全部依赖空行分段。
- 表格长单元格在移动端必炸：单元格超过一定长度就降级为列表。
- mermaid 渲染有成本，按内容 hash 缓存，Agent 反复改稿不会反复起 puppeteer。
- 幂等不等于 diff 干净：bullet 符号、强调符号这类序列化选项必须写死在代码里，否则每次 diff 全是噪音。

## 可复用建议

- 把 canonical markdown 当内部 IR，适配器是纯函数。新增平台 = 新增一个文件加一份 quirks 配置，别在主干里散落 if。
- 维护一组固定的语料样例做快照测试，模型升级后先跑语料再上管线。
- Agent 和人共用同一条发布路径，这是整条管线里最值钱的一条：单一事实源比任何格式技巧都重要。

## 总结

AI 生成解决的是"写"，多平台发布难的是"适配"。一条以 mdast 为核心、规范化前置、适配器后置的管线，能把格式问题收敛到几十行转换代码里。它不性感，但让 Agent 产出可以直接进入发布流——在自动化内容工作流里，这是实打实的收益。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/c1b886dcdc66d9d5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/6fccddef99e33124.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/a60f83b21a88da18.png)

