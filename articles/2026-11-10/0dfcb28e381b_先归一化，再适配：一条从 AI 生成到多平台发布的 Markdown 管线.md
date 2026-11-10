---
title: 先归一化，再适配：一条从 AI 生成到多平台发布的 Markdown 管线
feedId: 44018
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景

社区里越来越多的帖子由 Agent 起草：MCP 拉素材，模型出初稿，人做终审。写完只是第一步——同一篇 Markdown 要发 GitHub、公众号、知乎、掘金，四处的“方言”各不相同，手工搬运既低效又容易走样。

## 问题

三层问题叠在一起：

1. **AI 输出本身不稳定**：标题层级随手写、代码块忘标语言、偶尔夹带内联 HTML、中西文之间没有空格。
2. **平台方言差异**：公众号不认 Markdown，粘贴即丢结构；知乎对外链有限制；mermaid、脚注、宽表格不是处处可用。
3. **适配逻辑散落在人脑里**：每个平台的坑靠谁发谁记得，没有可复现的路径。

## 做法

思路只有一句：**先归一化，再适配**。

**1. 定义规范子集（canonical dialect）**
约束 AI 只生成一个 Markdown 子集：全文唯一 h1、代码块必须标语言、禁止内联 HTML、列表符号统一、表格不超过四列、frontmatter 声明 title/tags/summary/cover。这条约束写进 agent 的 system prompt，事后清洗量直接减半。

**2. AST 归一化**
用 remark 把文档解析成 mdast，跑一组确定性 transform：标题层级重排、中英文之间补空格、剥离残留 HTML、相对路径图片换成绝对 URL。这层的输出是“规范 Markdown”，与平台无关。

**3. 平台适配器**
每个平台一个纯函数 adapter，输入 mdast，输出平台载荷：

- GitHub：基本原样 GFM；
- 掘金 / 知乎：序列化回 Markdown，直接粘贴可用；
- 公众号：mdast → 带内联样式的 HTML，代码高亮在生成阶段就烧进 style 属性。

适配器只做格式转换，不改内容。

**4. 发布闸门**
自定义 markdownlint 规则 + 每个适配器的快照测试，CI 不过不发。发布产物落盘留档，改动可 diff。

**5. 接入 OpenClaw**
把管线封装成 MCP 工具或 skill：Agent 只负责往 `drafts/` 写文件，`publish(platform)` 触发后续流程。模型从此不接触格式细节。

## 踩坑点

- **公众号必须先渲染再复制**：直接贴 Markdown 是纯文本，class 名会被全部剥掉，只有内联 style 能活下来。
- **知乎正文外链**：可能被吞或不可点，重要引用转脚注，或者放“阅读原文”。
- **表格在移动端必炸**：列数上限写进规范子集，超限的由 transform 自动降级成列表。
- **图片路径**：相对路径只在仓库里有效，发布前统一换绝对 URL 或先上传到图床。
- **最大的坑**：让模型直接输出“公众号版”“知乎版”。格式必须由确定性代码生成，模型只产出规范子集。

## 可复用建议

- **单一事实源**：一篇规范 Markdown，所有平台产物都是派生物，改稿只改源头，重跑管线即可。
- **适配器保持纯函数**：输入输出都是字符串，快照测试一行搞定，平台规则升级时成本最低。
- **两道防线管同一条规范**：归一化规则沉淀成配置文件，与 prompt 里的生成约束同步维护，不要各写一套。

## 总结

把格式适配从“发布时的手工动作”前移成管线里的确定性步骤：AI 负责内容，规范子集负责收敛，AST transform 负责清洗，适配器负责方言。发布从此可复现、可测试、可交接。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/dac110938a701e59.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/2de4abed44747ea1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/b3a4672be73ed928.png)

