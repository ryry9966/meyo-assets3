---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 44048
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景与问题：上下文才是稀缺资源

Agent 用久了，常见的瓶颈不是模型"能力不够"，而是"能力太多"。系统提示词越写越长，几个 MCP server 一口气注册几十个工具 schema，结果模型注意力被稀释：选错工具、忽略关键指令、把简单任务复杂化的概率肉眼可见地上升。

问题的本质是**全量加载**——不管当次会话需不需要，所有能力说明都塞进上下文。Skills 是 OpenClaw 提供的折中方案：把一类能力打包成独立目录，平时只在上下文里占一小段元数据，命中场景时才真正展开。核心思想是渐进式披露（progressive disclosure）。

## 做法：一个 Skill 的最小结构

```
pdf-report/
├── SKILL.md        # 必需：frontmatter + 指令正文
├── scripts/        # 可选：确定性脚本
└── references/     # 可选：详细参考资料
```

`SKILL.md` 的 frontmatter 只有两项：`name` 和 `description`。加载分三层：

1. **启动时**：只有 name + description 进入上下文，单个 skill 大约几十 token；
2. **触发时**：会话内容命中 description 描述的场景，模型才去读正文；
3. **执行时**：正文里引用的参考文件和脚本，按需读取或运行。

落地步骤：

1. 建目录写 `SKILL.md`，description 里明确写触发条件，比如"当用户要求解析或生成 PDF 报表时使用"；
2. 正文控制在 500 行以内，版本差异、字段说明这类细节拆到 `references/`；
3. 格式转换、固定校验这类确定性操作写成脚本放进 `scripts/`，让模型调用而不是每次现场写代码；
4. 在项目配置里启用，跑几个真实会话验证触发是否准确。

## 踩坑点

- **description 写得太含糊**，skill 永远不触发；太宽泛，则无关场景误触发。直接写"何时使用"，不要写功能介绍。
- **一个 skill 塞多个职责**。触发判断靠 description 完成，职责越多边界越模糊。一个 skill 只做一件事。
- **以为模型会自动读完 `references/` 下所有文件**。不会。必须在正文里明确指示，例如"处理批量导入前，先读 references/schema.md"。
- **Skill 数量过百后**，description 列表本身成了新的上下文负担。按项目裁剪启用范围，真正通用的能力才放全局。
- **同名覆盖**：项目级 skill 优先于全局级，排查"为什么不生效"时先确认加载来源。

## 可复用建议

- 把 skill 当代码管理：进版本库、走 review，description 改动在 commit 里说明理由。
- 团队里反复粘贴的那几段 prompt，是最适合第一批 skill 化的对象，迁移成本最低、收益最直接。
- 判断类内容用自然语言写清楚，确定性步骤交给脚本，两者不要混在一起。
- 触发验证用真实会话，不要靠想象。每次改完 description，重新跑一遍典型问法。

## 总结

Skills 解决的不是"让模型会更多东西"，而是"让模型在对的时刻只看到对的东西"。它的价值不在于单个 skill 写得多漂亮，而在于这套按需加载的纪律：元数据小、触发准、细节延迟展开。把这套习惯建立起来，Agent 的上下文会干净很多，行为也更可预测。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/395a35776ac2c535.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/765a019911165ce4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/1b36bef6f589571f.png)

