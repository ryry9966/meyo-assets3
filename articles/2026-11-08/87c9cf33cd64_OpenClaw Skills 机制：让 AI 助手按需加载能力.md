---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 43849
source: 综合讨论
publishedAt: 2026-11-08
---

# 背景

OpenClaw 的 agent 能力越挂越多，prompt 也越来越胖。传统做法是把所有工具的 schema 全量塞进上下文：MCP 挂十个 server，光工具描述就可能吃掉几千 token，还让模型在几十个相似工具之间犹豫。Skills 是 OpenClaw 给出的另一种解法：能力不常驻，按需加载。

# 问题

我们想要的效果是：agent 平时轻装上阵，遇到特定任务才"翻出"对应的手册。拆开是三个工程问题：

1. **上下文成本**：所有能力的元信息常驻，正文按需读取。
2. **可用性判定**：本机缺依赖时，能力应自动隐藏，而不是运行时报错。
3. **分发与隔离**：社区技能和自研技能要分开管理，并按 agent 角色裁剪。

# 做法

一个 Skill 就是一个文件夹加一份 SKILL.md：

```markdown
---
name: pdf-extract
description: Use when the user asks to extract text or tables from PDF files.
metadata:
  requires:
    bins: [pdftotext]
---
（正文：步骤、命令、注意事项）
```

加载分两层：启动时只有 `name` 和一行 `description` 进入系统提示；模型判断当前任务命中后，才读取完整正文——即渐进式披露（progressive disclosure）。

实操步骤：

1. 写 SKILL.md，放在 `~/.openclaw/skills/<name>/`（全局），或工作区 `skills/` 目录（项目级，优先级更高）。
2. `openclaw skills list` 查看加载状态与合格性；`skills info <name>` 看单个详情。
3. 外部能力走 `openclaw skills install <pkg>` 从 ClawHub 安装，`skills update` 负责升级。
4. 在 agent 配置里按角色启用/禁用，控制每个 agent 可见的技能面。

# 踩坑点

- **description 是"选择器"，不是简介。** 写"处理 PDF"不如写 "Use when the user asks to extract text from PDF"。描述含糊，该触发时不触发，这是最常见的失效模式。
- **requires 门槛是双刃剑。** 本机没装 `pdftotext`，skill 会被判为不可用，模型根本看不到它。排查"为什么没用上"，先跑 `skills list` 看合格性。
- **workspace 覆盖全局。** 同名 skill 项目级优先，调试时容易改错文件，改了半天生效的是另一份。
- **正文别贪长。** SKILL.md 一旦命中会被整段读入上下文，几百行的手册等于把省下的预算又吃回去。大而全的技能应拆子文档、按需引用。
- **与 MCP 工具重复时模型会绕路。** 同一能力二选一：需要结构化 API 调用用 MCP，教模型用现有 CLI 用 skill。

# 可复用建议

- 把 skill 当**内部 SOP 文档**写，而不是当插件：触发条件、前置依赖、验证步骤三段齐全。
- description 用 "Use when ..." 句式，一个 skill 只做一件事，名字用动词域。
- 给不同角色的 agent 配技能 allowlist：网关 agent 和编码 agent 的技能面不应该一样。
- 社区技能升级后 diff 一下 SKILL.md，正文变化不会有任何通知。

# 总结

Skills 的价值不在"又一个插件系统"，而在于把上下文预算从"能力清单"转移到"当前任务"。元信息常驻、正文按需、依赖门槛自动过滤——这三件事组合起来，agent 才能既多能又不臃肿。建议从自己工作流里最重复的一个环节开始写第一份 SKILL.md，比一口气装十个社区技能更能理解这套机制。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/67ecd34e7e2ac4f8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/397e58dfbd9f130b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/5115ec612c8fea59.png)

