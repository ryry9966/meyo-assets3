---
title: 一套 Markdown 管线：让 Agent 生成的内容一次写、处处发
feedId: 43739
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

我们的内容工作流里，初稿基本由 Agent 生成：在 OpenClaw 里跑一个写作 Agent 产出 Markdown 草稿，人工审校后要同步到三四个渠道——内部静态站、GitHub 仓库、掘金、公众号。“写一次、发多处”是老需求，但 Agent 参与生成之后，问题形态变了：生成端不稳定的格式，会被管线原样放大到每一个发布端。

## 问题

直接把 AI 生成的 Markdown 投到各平台，几乎每步都掉坑：

- 各平台 Markdown 方言不一：脚注、任务列表、公式、表格对齐，支持度参差；
- 公众号根本不吃 Markdown，只接受带内联样式的受控 HTML；
- AI 生成内容自带“方言”：标题层级跳跃、代码块语言标签随手写、偶发裸 HTML 和 `$$...$$` 公式；
- 图片是最脆的一环：相对路径、外链失效、平台图床互不认。

逐平台手工修，等于每发一篇就把成本乘以平台数。

## 做法

思路是把“适配”从发布动作里拆出来，做成显式管线，核心是统一中间表示 + AST 级转换：

1. **规范源**。定义一份内部 Markdown 子集（CommonMark + 受限 GFM），连同 front matter 元数据作为唯一事实源。把这份约束直接写进写作 Agent 的 system prompt，生成端就开始收敛。
2. **Lint 关口**。发布前先过 markdownlint，规则按子集裁剪：禁裸 HTML、禁相对路径图片、标题从 h2 起、代码块必须带语言标签。注意 Lint 只对规范源做，转完再查就晚了。
3. **转换层**。用 unified/remark 走 AST，不碰正则：remark-parse 解析 → 自定义插件做平台差异改写（脚注降级为括号引用、公式转图片或删减、图片链接统一替换为图床地址）→ remark-rehype 出 HTML。
4. **平台渲染器**。一个目标一个 renderer：静态站直接吃规范源；掘金基本直发；公众号走内联样式模板，用 headless 浏览器渲染一遍再校验。
5. **发布工具化**。以上每步封装成 MCP 工具——`lint_md`、`convert`、`upload_image`、`publish`——由 Agent 编排，人只在审校和最终确认介入。

## 踩坑点

- 别用正则做格式转换。Markdown 的嵌套结构（列表里的代码块、引用里的表格）正则必翻车，AST 才稳。
- front matter 忘了剥就贴到掘金，会被渲染成一张表。
- 代码块语言标签要建白名单映射：AI 常把 `text`、`shell`、`plaintext` 混着写，部分平台高亮直接失效。
- 公众号对标签和样式白名单比想象中窄，内联转换后必须渲染快照验证，不能只信转换器的“成功”返回。
- 每个 target 留 golden file 快照，转换器升级后 diff 一下，格式回归一眼可见。

## 可复用建议

- 永远只改规范源，平台变体是构建产物，不要手工编辑；
- 把 Markdown 子集约束前移到生成端 prompt，比在转换层兜底便宜得多；
- 图片在管线早期就收敛到稳定 URL，越晚处理越乱；
- 一个关注点一个 MCP 工具，方便单测和替换。

## 总结

多平台发布难的不是“转格式”，而是让格式决策只发生一次。约束前移到生成端，差异后移到 AST 转换层，执行收敛成可编排的工具——这条管线跑顺之后，新增一个发布渠道的边际成本，基本只剩写一个 renderer。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/ca97337ef6d59f20.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/5edbbc834f78ec8c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/9bdb5789c3318c69.png)

