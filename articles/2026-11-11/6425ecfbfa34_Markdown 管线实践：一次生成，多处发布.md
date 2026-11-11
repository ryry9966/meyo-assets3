---
title: Markdown 管线实践：一次生成，多处发布
feedId: 44103
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景

用 OpenClaw 跑内容类 Agent，普遍会遇到同一个问题：模型生成 Markdown 很快，发出去很慢。公众号、知乎、掘金、博客各吃各的方言，人工粘贴一次内容就变形一次。我们最后把这件事做成了管线，这里记录设计取舍。

## 问题

多平台发布的差异集中在四类：

1. **方言差异**：公众号不认 GFM 表格和外链锚文本；知乎对 HTML 过滤激进；静态站需要 front-matter。
2. **资源差异**：图床各有防盗链，公众号图片必须重新上传。
3. **代码块差异**：高亮方案、行号、复制按钮各平台实现不同。
4. **状态差异**：哪些平台发过、发的是哪个版本，没人说得清。

人工处理这四件事，每篇稿子 30 分钟起步，还容易漏。

## 做法

管线分五段，核心原则是「单一事实源 + 平台适配器」：

**第一步：归一化。** Agent 产出的 Markdown 先过 cleaner：标题层级收敛（H1 唯一）、剥掉思考痕迹、补全 front-matter（title/slug/tags/cover/platforms）。用 remark 系 AST 处理，别用正则硬抠。

**第二步：lint 门禁。** 发布前跑规则：图片必须是可下载 URL、代码块闭合、表格列数一致、分割线和加粗密度不超标。不通过就打回给 Agent 自修，而不是人工兜底。

**第三步：适配器转换。** 每个平台一个纯函数 adapter：

```js
// 输入 canonical AST，输出平台 payload
async function wechatAdapter(doc) {
  const html = await renderWithTheme(doc);   // 套公众号内联样式
  const images = await reuploadImages(html); // 换成公众号图床
  return { html, thumb: doc.cover, digest: doc.summary };
}
```

公众号走「Markdown → 内联样式 HTML」；掘金、知乎走各自 API 的 Markdown 原文；静态站只补 front-matter 渲染。

**第四步：发布执行。** 发布动作封装成 MCP tool，Agent 只负责调用，管线负责鉴权、限速和重试。

**第五步：状态回写。** 每次发布把 platform、post_id、content_hash 写回 sidecar 文件。hash 一致就跳过，保证幂等。

## 踩坑点

- **三个反引号会提前闭合**：AI 在代码块里讲 Markdown 语法时最容易踩。cleaner 里统一检测嵌套围栏并升级为四反引号。
- **防盗链是发布后才暴露的问题**：本地预览正常，发到知乎后原图站挂了就全裂。管线里一律先下载再转存，别信外链。
- **front-matter 被渲染成正文**：部分平台把 `---` 块当文本显示，适配器必须在发送前剥掉。
- **判重靠 hash 不靠标题**：改过标题的稿子用标题判重会重复发布，content_hash 才可靠。
- **LLM 喜欢加装饰**：大量 `---`、整段加粗、每句一个 emoji。在 lint 里设阈值，比在 prompt 里求它收敛有效得多。

## 可复用建议

1. canonical 文档永远只有一份，平台产物一律视为缓存，可随时重建。
2. adapter 写成纯函数 + 快照测试，平台 API 变了只改一处。
3. lint 规则和 prompt 约束配套写：生成端少犯错，管线端兜底。
4. 状态文件进 git，发布历史可追溯、可回滚。

## 总结

这件事的本质不是「格式转换」，而是把发布从手工动作变成可测试的构建过程：归一化保证输入干净，适配器隔离平台差异，状态回写保证幂等。做到这三层，Agent 产出的内容才能真正一次生成、多处发布，而且出了问题能定位是哪一层的锅。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/b86cffd444e87155.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/8dfabff10051c420.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/7f964128d48e2bd3.png)

