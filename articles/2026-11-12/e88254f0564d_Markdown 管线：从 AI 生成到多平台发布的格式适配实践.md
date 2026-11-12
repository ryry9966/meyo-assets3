---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配实践
feedId: 44241
source: 综合讨论
publishedAt: 2026-11-12
---

## 背景

用 Agent 产出内容时，Markdown 是事实上的中间格式：模型擅长写、diff 好审、Git 好管。但“写完”和“发出去”之间隔着一段脏活——公众号不认表格和脚注，知乎会剥离大部分 HTML，静态站要 frontmatter，GitHub 只认 GFM。同一份稿子发三端，往往要手工改三遍。

## 问题

手工适配有三个代价：结果不可复现，每次微调都不一致；结构容易丢——AI 产出里常见的嵌套列表、粗体当标题、裸 HTML 片段，到了不支持的平台上直接塌掉；而且 AI 输出本身不稳定，换模型或版本，Markdown 方言就有偏差。管线化的目标就是把这些不确定性收敛掉。

## 做法

思路是拆成“规范化 → 校验 → 适配 → 渲染”四段，中间用 mdast（unified/remark 生态）做唯一中间表示，全程不碰正则。

1. **规范化**：Agent 只落一个 canonical.md，不直接产目标格式。入口先跑 remark 标准化：统一列表缩进、压平多余转义、补 frontmatter 默认值，保证源头干净且幂等。
2. **校验**：用 remark-lint 定制规则约束 AI 输出——标题层级必须连续、禁裸 HTML、表格单元格限长、禁脚注。不过就打回给 Agent 重写，而不是让下游静默降级。这一步挂进 Agent 工具循环，模型可以自纠。
3. **适配**：每个平台一个 remark 插件作 adapter，输入输出都是 mdast，行为由 profile 配置驱动：

```yaml
# profiles/wechat.yaml
tables: list      # 表格降级为加粗行列表
footnotes: appendix
images: upload_and_rewrite
html: strip
```

4. **渲染**：md→HTML 后注入内联样式（公众号只认 inline style），GitHub/静态站保留原生 md。产物全部写进 dist/对应平台/，canonical 永远是唯一真源。

整条链就是一个 CI hook：提交 canonical.md → 校验 → 各 adapter 并行出 dist → 人工只审 diff。

## 踩坑点

- 公众号屏蔽外链图片，必须走平台上传接口回填 CDN 地址，这步没法纯本地完成，要预留半自动环节。
- 流式输出期间别解析，半截 Markdown 会碎。要么等完成信号，要么用容忍残缺语法的解析器。
- 转义是重灾区：代码块里夹反引号、双重转义会让渲染层输出字面斜杠，规范化阶段统一压平一次。
- 别把适配逻辑写死在代码里。平台规则常变（尤其公众号），改 profile 配置比改代码便宜得多。
- 幂等断言必须有：管线跑两遍输出必须一致，否则迟早二次变形。

## 可复用建议

- 用 AST 不用正则，mdast 上改节点比字符串替换可靠一个量级。
- 每个平台配 golden file 快照测试，规则变更时肉眼可审。
- 校验前置于生成：lint 规则同时写进 Agent 系统提示和工具循环，双保险。
- dist 不进版本库，避免有人手改 dist 导致真源漂移。

## 总结

这套管线的核心不是格式转换本身，而是隔离两个变量：AI 输出的不确定性，用“源头标准化 + 校验”收口；平台的不兼容，用“AST 中间表示 + profile”收口。做到这两点，多平台发布就从手工活变成可测试、可回归的工程问题。OpenClaw 用户可以直接把校验和 adapter 挂进现有的 Agent 工具链，增量接入即可，不必一步到位。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/8de6d82f15fe05a9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/4161623656e43a41.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/4e5be8f004d5c4bb.png)

