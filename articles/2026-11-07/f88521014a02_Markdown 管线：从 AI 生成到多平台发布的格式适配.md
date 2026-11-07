---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 43741
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

技术帖的发布渠道越来越多：自建站、GitHub、掘金、公众号、知乎。内容源头基本是 Agent 生成或半自动生成的 Markdown。"写一次、发多处"听起来简单，真做起来每个平台都要手动调一遍格式，改到第三遍就会想写脚本。

## 问题

AI 生成的 Markdown 表面合规，实际混着各种"方言"：标题层级随机（有时直接从 `###` 开始）、代码围栏缺语言标注、列表缩进 2 空格和 4 空格混用、文末留一堆 reference 式链接定义。这些在 GitHub 上无所谓，到公众号或知乎的渲染器里就会碎掉。根因是没把**源稿**和**平台稿**分开，也缺一个统一的归一化环节。

## 做法

我们把管线拆成四层，全部跑在 unified/remark 生态上，MDAST（Markdown AST）作为中间表示：

**1. 归一化层**。对 AI 原始输出做确定性清洗：标题统一降到 H2 起步、代码围栏补 language tag、列表缩进统一为 4 空格（对宽松渲染器最安全）、剔除 AI 偶尔吐出的行内 HTML。产出"规范源稿"入库 Git，作为唯一事实来源。

**2. 校验层**。基于 remark-lint 加自定义规则：禁用 reference 式链接（部分平台会吞掉文末定义）、图片必须是图床绝对路径、外链做 HEAD 请求验活，死链直接拦截。

**3. 转换层**。每个平台一个 adapter，输入 MDAST 输出目标格式：公众号走 remark-rehype 转 HTML 再内联样式；掘金/知乎基本原样输出 GFM，只删 front-matter；自建站保留 front-matter 喂 VitePress。核心原则：在 AST 上做变换，不要拿正则改字符串——正则改 Markdown 十次有三次会误伤代码块。

```js
// adapter 示意：宽表格在公众号渲染差，超阈值转图片
visit(tree, 'table', (node, i, parent) => {
  if (countColumns(node) > 4) {
    parent.children[i] = imageNode(renderTableToPng(node));
  }
});
```

**4. 发布层**。GitHub Actions 串起来：PR 里跑 normalize + lint，合并后按标签分发到各 adapter，每个平台生成 preview diff，人工确认后再推。全自动发布我们刻意没开——平台渲染差异太大，留一道人工检查值得。

这套管线后来封装成一个 MCP tool，Agent 写完稿直接调用，人只看 diff。

## 踩坑点

- front-matter 的 `---` 会被部分渲染器当成水平线，删除时要连带校验内容，别只做字符串匹配。
- 列表项内嵌代码块时，缩进差一格整个列表就散架。归一化必须显式处理嵌套结构，只处理顶层不够。
- AI 会"幻觉"出不存在的链接，外链验活这步不能省。
- 表格在公众号渲染极差：我们按列数阈值自动分流，宽表转 puppeteer 截图，窄表降级为列表。
- 正文里的裸 `<tag>` 会被部分平台当 HTML 吃掉，转换层统一做实体转义。

## 可复用建议

- 源稿进 Git，平台稿一律视为构建产物，禁止手改。
- 所有格式操作走 MDAST，正则只做最后兜底。
- adapter 保持纯函数：同输入必出同输出，方便 diff 回归测试。
- 每个平台维护一份"已知坑"清单，新 adapter 上线前先过一遍。

## 总结

这条管线真正解决的不是"转格式"，而是把 AI 输出的不确定性关进一个确定性环节。归一化做在前面，后面的校验、转换、发布才都成立。全部代码不到四百行，收益是单篇内容发布从半小时手动调整降到两分钟确认 diff。如果你也在多处分发 AI 生成的内容，建议先把归一化层做出来——它是整条管线里性价比最高的一段。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/a8fdf982c9a46172.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/7548266a9725bb85.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/cbf9cb4280e659d4.png)

