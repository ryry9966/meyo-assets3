---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 43363
source: 综合讨论
publishedAt: 2026-11-03
---

# Markdown 管线：从 AI 生成到多平台发布的格式适配

## 背景

用 Agent 产出内容已经不是新鲜事：写教程、整理周报、生成文档。但真正的麻烦在"最后一公里"——同一段 Markdown，要同时投到博客、公众号、社区论坛、Notion，各家的格式规则并不兼容。我们的解法是把格式适配从人肉复制改成一条可复现的管线。

## 问题

实践中踩到三类问题：

1. **AI 输出本身不规范**：LLM 喜欢输出嵌套列表、单个换行的软分段、中文弯引号、错误的代码块语言标签，偶尔还夹带 HTML。
2. **平台差异**：公众号不支持锚点跳转、部分编辑器不认 mermaid、GitHub 和静态站的 heading 锚点 ID 生成规则不一致。
3. **资产引用**：本地图片路径、相对链接到发布端全是死链。

靠正则打补丁，平台规则一变就崩一次。

## 做法

管线分四层，全部基于 mdast/AST 操作，不用正则硬匹配：

```
AI 生成 → 规范化 normalize → 按平台 transform → 资产上传 → 发布校验
```

1. **规范化层**：所有来源先过 lint。统一换行（`\n` → `\n\n` 分段）、剥离 frontmatter、折叠过深嵌套、弯引号转直引号、修正 fence 语言标签。产出"内部规范 Markdown"。
2. **变换层**：每个平台一个 remark 插件，纯函数转换。公众号目标：外链转文末引用、mermaid 降级为预渲染 SVG、样式内联；博客目标：相对图片路径换 OSS 绝对地址、按平台规则生成锚点。
3. **资产层**：图片上传封装成独立 MCP 工具，上传后回填 URL，本地路径只作缓存键。
4. **发布层**：各平台 API 封装为 MCP tool，输入是转换产物而非原始 Markdown。发布前强制 dry-run，输出 diff 供人工确认。

骨架示意：

```js
const doc = normalize(await read(source));
for (const target of targets) {
  const out = await transform(target, doc); // 纯函数
  await uploadAssets(out);
  await publish(target, out, { dryRun: true });
}
```

## 踩坑点

- **锚点 ID**：GitHub 保留中文并用连字符拼接，公众号干脆不支持。目录必须按目标平台分别计算。
- **换行语义**：CommonMark 里单换行是软换行，微信编辑器会渲染成 `<br>`。AI 输出大量单换行，不处理则排版稀碎。
- **mermaid**：别赌平台支持，统一在管线里预渲染成 SVG 再上传。
- **防盗链**：公众号会屏蔽外链图片，要么传素材库，要么转 base64（注意体积上限）。
- **幂等性**：发布重试时，transform 不幂等会产生重复的文末链接列表。所有插件必须"输入相同则输出相同"。
- **正则地狱**：凡用正则处理 Markdown 的地方最后都改成了 AST 操作。mdast 多花半天，省掉后面无数补丁。

## 可复用建议

- 单一信源 + 多目标构建，绝不维护多个版本的内容。
- 元数据（目标平台、封面、标签）放 frontmatter，管线读取，不靠人工传参。
- 每个 transform 配快照测试："规范输入 → 期望输出"，升级依赖时回归。
- 永远 dry-run + diff 在前，API 发布在后。
- 规范化层抽成独立插件包，多个项目共享。

## 总结

这条管线的本质不是"转换格式"，而是把格式规则显式化、工程化。AI 负责生成内容，管线负责让内容在任何平台以预期形态出现。顺序建议：先建规范层，再做平台插件，最后接发布 API——顺序反了，就会陷入无止境的正则修补。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-03/3f76402c39d233c3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-03/6738d935dccb5797.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-03/ee67dae6e0f74856.png)

