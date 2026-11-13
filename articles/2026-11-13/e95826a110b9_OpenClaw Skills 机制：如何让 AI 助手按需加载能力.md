---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 44343
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

随着接入的工具和 SOP 越来越多，很多人把所有能力描述、操作手册一股脑堆进 system prompt。结果上下文轻松突破几万 token：推理变慢、费用上升，模型在几十个相似工具之间反复选错。OpenClaw 的 Skills 机制就是针对这个矛盾：能力按目录组织，启动时只加载"索引"，运行时按需展开。

## 问题

三个具体矛盾：

1. **能力数量 vs 上下文预算**：50 个技能的完整说明可能占 3~5 万 token，不可能常驻。
2. **描述粒度 vs 触发准确率**：写得含糊，模型想不起来用；写得宽泛，又到处误触发。
3. **确定性步骤 vs 自然语言**：让模型"读文档再照做"，不如让它直接执行脚本。

## 做法

核心是三级渐进加载（progressive disclosure）：

1. **索引常驻**：每个技能是一个目录，`SKILL.md` 的 frontmatter 写 name 和 description。OpenClaw 启动时扫描技能目录，只把 name + description 注入 system prompt，单条控制在两三行。200 个技能的索引也就几千 token。
2. **正文按需展开**：模型判断任务命中某技能时，才把 `SKILL.md` 正文（操作步骤、约束、输出格式约定）载入上下文。
3. **资源引用**：正文可引用 `scripts/`、`references/` 下的文件。确定性流程写成脚本直接执行——脚本本身不进上下文，只有输出结果进。

目录结构示例：

```text
skills/
  pdf-extract/
    SKILL.md                 # frontmatter: name, description
    scripts/extract.py
    references/layout-spec.md
```

写作要点：

- description 用 "Use when ..." 句式写触发条件，并写反例（如"不要用于图片 OCR"）。
- 正文用祈使句写步骤，明确输入、输出字段和失败兜底。
- 脚本不依赖 CWD，用相对 `SKILL.md` 的路径解析资源。
- 用 `openclaw skills list` 查看索引体积，按 workspace 灰度启停。

## 踩坑点

- **description 是检索键，不是简介**。写成"处理 PDF 相关任务"，命中全靠运气。触发场景、输入形态、排除条件都要点明。
- **正文写成散文**。模型要的是可执行步骤：编号、明确 JSON 字段名，比一段流畅的背景介绍有用得多。
- **技能互相引用成环**。A 指向 B、B 又指回 A，上下文来回膨胀，保持单向引用。
- **脚本权限**。技能脚本随 agent 权限运行，务必放进沙箱或受限目录，别让脚本能碰到凭证路径。
- **只写不测**。准备 10~20 条典型触发语句做回归，统计误触发和漏触发，比肉眼审 description 有效。

## 可复用建议

- 一个技能只做一件事，命名用动词短语（pdf-extract、log-triage）。
- 确定性流程下沉到脚本，脚本输出务必结构化；需要判断的部分才写在正文里。
- 已有 MCP 工具不必搬进技能：技能正文负责"何时调、按什么参数调"，MCP 工具负责执行，二者是编排关系不是替代关系。
- 技能进 git，review 照代码流程来；description 与正文的改动放同一次提交。
- 上线后记录触发日志：哪些技能被加载、加载后任务是否成功。两周数据足够决定砍掉哪些。

## 总结

Skills 本质上是给 agent 能力做的"按需分页"：索引常驻、正文换入、资源引用。把 description 当检索键打磨，把正文当操作手册写，把确定性逻辑下沉到脚本，上下文预算和触发准确率就能同时保住。如果现在的 system prompt 里还压着几段大 SOP，先挑最占篇幅的三段拆成技能试一下，效果最直观。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/3cfdefb2646e13ef.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/24d4f4be2b9f963c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/0e2809c7d68f712f.png)

