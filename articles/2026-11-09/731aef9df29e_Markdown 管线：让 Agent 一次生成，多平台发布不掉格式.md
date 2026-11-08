---
title: Markdown 管线：让 Agent 一次生成，多平台发布不掉格式
feedId: 43867
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

用 OpenClaw 或其他 Agent 写技术稿，生成环节已经很成熟：给个大纲，模型吐出一份结构尚可的 Markdown。真正的摩擦在下游——公众号编辑器吞表格、知乎不认行内 HTML、掘金和静态站对代码高亮的处理各不相同。一份稿子手动适配三个平台，往往比写稿本身还费时间。

## 问题

本质上是三件事没分开：

1. **生成不受约束**：模型随手混用原生 HTML、脚注、Frontmatter，输出形态不可控；
2. **没有中间表示**：文本被当字符串处理，正则替换叠正则替换，改一处崩一片；
3. **平台差异散落各处**：适配逻辑写在发布脚本里，没有抽象成可插拔的转换层。

## 做法

我的管线核心是“一份源 Markdown + N 个纯函数适配器"，基于 unified/remark 生态。

**第一步：约束生成。** 在 Agent 的 system prompt 里写一份样式契约：只用标准 Markdown 语法，图片一律用占位符 `{{img:slug}}`，禁止行内 HTML 和脚注。生成完立刻跑 markdownlint，不合格打回重生成，比事后修补便宜得多。

**第二步：解析为 AST。** 用 remark 把 Markdown 解析成 mdast，后续所有操作都在语法树上做，不碰原始字符串。

**第三步：平台适配器。** 每个适配器是一个 `mdast -> mdast` 或 `mdast -> html` 的纯函数：

```js
// 公众号适配器（示意）
const wechat = (ast) => visit(ast, 'code', (node) => {
  node.type = 'html';
  node.value = inlineStyleHighlight(node); // 高亮转行内样式
});
```

公众号输出带行内样式的 HTML（编辑器只认这个）；知乎、掘金、静态站基本直接序列化回 Markdown，只做小修正（表格列宽、任务列表降级）。

**第四步：图片独立处理。** 下载源图 → 按平台 API 重传 → 替换 URL。这一步必须独立于格式转换，因为公众号有防盗链和素材接口限制，重试逻辑很啰嗦。

**第五步：发布前校验。** 对产物做断言：无外部图片域名、代码块都已转行内样式、无残留占位符。校验失败不发布。

我把整条管线包成 MCP tool 暴露给 Agent，参数就一个 `platform`，Agent 写完稿自己就能触发发布。

## 踩坑点

- **别用正则改格式**。嵌套列表、代码块里的 `|` 和 `<`，正则几乎必翻车，mdast 上操作稳得多。
- **适配器要幂等**。跑两遍不应双重转义，用 golden file 快照做回归测试。
- **LLM 会"帮忙"输出 HTML**。哪怕约束过的 prompt 也有漏网，lint 兜底不能省。
- **公众号图片是重灾区**。外链图会被吞，必须走素材接口重传，还要处理 GIF 的尺寸限制。
- **序列化往返有损**。mdast 转 Markdown 再解析，紧凑列表等边界结构会变形，尽量一转到底、不回头。

## 可复用建议

- 样式契约写成独立文件，生成端和校验端共用，别散在 prompt 里；
- 适配器保持无状态、可组合，新平台半天能接上；
- 先跑通一个平台再横向复制，别一上来铺五个。

## 总结

多平台发布难的不是 Markdown 本身，而是缺一层稳定的中间表示。把“生成约束、AST 转换、独立图片管线、发布校验”四段拆开，每个平台退化成一个小适配器，Agent 的产物才真正做到一处生成、处处可用。欢迎在评论区交流你们平台的适配怪癖。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/5350771f7995d1df.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/b7b9b12b5ad7648c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/6c49e38a7ffa99c4.png)

