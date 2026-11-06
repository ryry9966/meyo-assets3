---
title: Markdown 管线：从 AI 生成到多平台发布的确定性适配
feedId: 43594
source: 综合讨论
publishedAt: 2026-11-06
---

## 背景

Agent 的产出量上来之后，瓶颈往往不是生成，而是分发。典型场景：Agent 写好一份 Markdown，要同时发博客（Hugo/Hexo）、公众号、知乎、掘金、GitHub README。各家格式约束不同，很多人第一反应是让模型"顺便转成 HTML"，结果既不稳定也无法审计，迟早翻车。

## 问题在哪

- AI 产出的 Markdown 不干净：脚注 `[^1]`、列表缩进混乱、代码块里夹 `**`、偶尔混入裸 HTML。
- 平台差异是硬约束：公众号只接受内联样式的 HTML，会剥掉 class 和 `<style>`；知乎/掘金只支持 Markdown 子集；静态站需要 frontmatter。
- 图片相对路径发布即失效。

结论：格式转换必须交给确定性代码。LLM 只负责产出规范 Markdown，转换归管线管。

## 做法：一份规范化 + N 个渲染器

核心是把"规范化"和"平台适配"拆开，用 unified/remark 全程走 AST：

```js
const tree = unified()
  .use(remarkParse)
  .use(remarkGfm)
  .parse(aiOutput); // AI 输出 → mdast
```

**1. 规范化**：在 AST 上统一处理——脚注转行内引用或删除、列表缩进归一为 4 空格、剔除白名单外的 HTML 节点、消除 `---` 与 frontmatter 的歧义。产出一份"规范 Markdown"作为唯一事实源。

**2. 校验**：自定义 lint 规则，如标题必须从 h2 起、图片必须有 alt、表格单元格内禁止代码块。不合规直接打回给 Agent 重写，别指望渲染器兜底。

**3. 平台适配**：每个平台一个纯函数渲染器。公众号走 remark-rehype，把代码块服务端高亮后展开成带 inline style 的 span；静态站在 AST 头部注入 frontmatter 再 stringify；知乎/掘金从规范 Markdown 做减法，删掉不支持的节点。

**4. 资源处理**：发布前先上传图片（OSS/图床 API），拿到 URL 后在 AST 上重写链接节点——改节点，不是对原文做字符串替换。

**接进 Agent**：整条管线封装成 MCP tool，如 `publish_markdown(platform, dry_run)`。Agent 只调工具不做转换；`dry_run` 返回渲染 diff，先审后发。

## 踩坑点

- 别用正则处理 Markdown。我最初在文本层替换图片链接，把代码块里的示例一起改了；切到 AST 后此类问题归零。
- 脚注是重灾区：AI 很爱生成 `[^1]`，公众号渲染器轻则丢内容重则整段崩，规范化阶段统一转行内引用最省心。
- GFM 表格里塞行内代码，知乎会错位，校验规则直接禁掉，成本最低。
- 公众号代码块必须服务端展开成 inline style，任何依赖 class 的客户端方案都会被剥掉。
- 平台行为会变：渲染器带版本号，配几份真实产出的快照测试，平台改版跑一遍就知道坏了哪里。

## 可复用建议

1. 把 AI 输出当不可信输入：规范化 + lint 是管线的一部分，不是可选项。
2. 转换逻辑写成小纯函数（AST in → AST out），方便单测与快照对比。
3. 让模型做内容，让代码做格式——边界划清，管线就稳。

## 总结

多平台发布没有银弹，但"规范 Markdown 唯一事实源 + AST 化转换 + MCP 工具化"这套结构，能把格式适配从每次手改变成一次性工程投入。新增平台只需加一个渲染器，Agent 侧无感。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/6b6a9683ba3fa46c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/552efb2cde54ba88.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/d27a7981fb85c4cc.png)

