---
title: 把 Markdown 当中间表示：从 AI 生成到多平台发布的格式管线
feedId: 44181
source: 综合讨论
publishedAt: 2026-11-12
---

## 背景

用 OpenClaw 这类 agent 日常产出技术内容，典型链路是：模型生成 Markdown 草稿 → 审核 → 发布。坑都在最后一公里：Markdown 是标准，但落地目标不是。公众号编辑器、知乎、掘金、静态博客、RSS，各自支持的语法子集、图片与链接策略全不一样。稿子写完只是开始，格式适配才是重复劳动所在。我们在一次“一个 agent 同时维护博客和公众号”的实践里，把这块抽成了独立管线。

## 问题

差异集中在四类：

1. **语法子集**：公众号只吃内联样式的 HTML，外链会被吞掉；GFM 任务列表、脚注在部分平台直接不渲染。
2. **资源引用**：相对路径图片发出去必 404，图床选择又和平台审核策略纠缠。
3. **元数据**：front-matter、标签、摘要，字段名和格式各平台不同。
4. **模型输出不稳**：混入裸 HTML 标签、YAML 冒号不加引号导致 front-matter 解析失败、mermaid 和公式在目标平台不渲染。

如果每个平台手改一遍，自动化就失去意义。

## 做法

核心思路：**把 Markdown 当中间表示（IR），格式差异全部下沉到管线，不在 prompt 里解决。** 分四步：

**1. 生成契约。** 在 skill 说明里声明允许的语法子集：只用标准 Markdown，禁 HTML，单文档最多一个 H1，代码块必须标语言，元数据走固定 front-matter 模板。不指望模型 100% 遵守，只求把违规率压到 lint 兜得住的水平。

**2. lint 校验。** 用 remark-lint 加自定义规则做结构校验：裸 HTML、超三层嵌套列表、YAML 解析失败、非绝对 URL 图片，命中即打回给 agent 重写。fail fast 比人工擦屁股便宜。

**3. AST 转换。** 用 unified/remark 解析成 AST，每个目标平台一个纯函数 transformer。以公众号为例：链接节点统一改写成文末“参考链接”脚注；图片节点替换成占位符，上传拿回 CDN 地址后回填；最后经 rehype 序列化成内联样式 HTML。

**4. adapter 发布。** 每平台一个 adapter，输入转换后的 payload，输出发布结果，互相不依赖。整条管线封装成一个 skill 或 MCP 工具，agent 写完稿一次调用完成分发。

```js
const ast = remark().parse(md);
lintOrThrow(ast, contract);       // 校验
const payload = wxTransform(ast); // 平台转换
await publish('wechat', payload); // 发布
```

## 踩坑点

- 别指望 prompt 解决格式问题，模型会“偶尔”忘约定，可靠的是 lint + 重写循环。
- front-matter 是重灾区：标题里的冒号、中文引号都会炸解析。加引号，或干脆换 JSON。
- 公众号富文本粘贴和 HTML 注入行为不一致，自动化优先走草稿接口，别依赖模拟粘贴。
- 表格移动端必溢出，超 4 列在转换阶段降级为列表。
- mermaid/KaTeX 只在自家博客能渲染，发外部平台前要拦截或转成图片。
- 图片回填后记得再校验一遍，占位符漏改是最常见的静默事故。

## 可复用建议

- 维护一份“允许的节点类型”清单，作为生成契约和 lint 规则的单一事实来源。
- transformer 写成纯函数：AST 进，payload 出，不碰 IO，才能做快照测试。
- golden file 回归：改 transformer 时 diff 转换结果，防止改 A 平台弄坏 B 平台。
- 图片资产独立管理，正文只留 token 占位，上传与回填是独立步骤。
- 新平台适配从改一个现有 transformer 开始，比从零写快得多。

## 总结

多平台发布的复杂度不在写作，在格式。把 Markdown 定位成 IR，生成端管契约、入口管 lint、中间管 AST 转换、出口管 adapter，职责分开之后，新增平台只剩小工作量。对 agent 工作流来说，管线封装成一个工具调用，“写完即发布”才算真正闭环。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/908cf32c34ff0d46.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/cd9d1213615bb3ce.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/49c12a87f9853f93.png)

