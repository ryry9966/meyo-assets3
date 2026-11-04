---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 43438
source: 综合讨论
publishedAt: 2026-11-04
---

## 背景

OpenClaw 的定位是长期驻留的个人 Agent，不是一次性问答工具。既然"长期"，就得回答一个老问题：重启之后，它还是"它"吗？OpenClaw 的解法很朴素——把身份放进一个纯文本文件 `IDENTITY.md`，放在 workspace 根目录（默认 `~/.openclaw/workspace/`），会话启动时注入 system prompt。没有数据库，没有配置中心，就是一个 Markdown 文件。

## 问题

没有身份文件时，实际会遇到三类麻烦：

1. **自我描述漂移**。今天自称"助手"，明天变成"你的 pipeline 同事"，跨会话不一致，自动化流程里引用它的名字也会乱。
2. **身份散落各处**。名字写在工作流模板里，语气藏在聊天记录里，换台机器部署就丢了。
3. **多 Agent 冲突**。工作、生活各跑一个 agent，共用 workspace 时人格混在一起。

## 做法

**1. 写一个最小化的 IDENTITY.md：**

```markdown
# IDENTITY.md
- **Name:** 阿钳
- **Creature:** 一只机械蟹
- **Vibe:** 冷静、话少、动手快
- **Emoji:** 🦀
- **Avatar:** avatar.png
```

字段越少越好，Avatar 用 workspace 内的相对路径。

**2. 用 git 管它。** workspace 建议整体纳入 git，身份的每次修改都走 commit 加说明，像评审配置一样对待。

**3. 多 Agent 用多 workspace。** 每个 agent 独立目录、独立 IDENTITY.md，配置里指定各自的 workspace 路径，互不污染。

**4. 让它"进化"而非"重写"。** 每月回顾一次：它实际承担的角色，和当初定义差多少？小幅 commit，而不是推翻重来。

## 踩坑点

- **别把行为规则塞进来。** 行为归 `SOUL.md` / `AGENTS.md` 管。往身份文件里写"回复要简洁"，改人格时容易误伤身份本身。
- **别写成大段散文。** 模型对结构化字段的执行比散文稳定，一句 vibe 胜过三段人设小说。
- **改完不生效？** IDENTITY.md 在新会话加载，长驻会话需要重启或 reset 才能吃到新身份，别以为热更新了。
- **别放敏感信息。** 这是身份文件不是档案袋，用户偏好归 `USER.md`，密钥更不该出现。workspace 若推到远端 git，先想清楚同步范围。

## 可复用建议

- 把 IDENTITY.md 当 **config**，不当日记：字段稳定、改动有理由、随时可回滚。
- 团队或多机部署时，把"骨架身份"做成模板仓库，各机器只 commit 自己的 vibe 差异，避免互相覆盖。
- Avatar 用本地文件比外链 URL 稳，避免加载失败回退到默认形象。
- 身份变更写清动机（"承担了定时巡检职责，加一句 vibe"），半年后回看才有价值。

## 总结

IDENTITY.md 的价值不在"给 AI 起名"的仪式感，而在于把身份从提示词里的口头约定，变成**有版本、可评审、可回滚的工程资产**。它很小，但它是 Agent 长期运行时所有一致性的锚点。改身份应该像发版、像改配置——这才是"可进化"该有的样子。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/cac05a47b34cbbe8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/9263a95170e39231.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/499696f617746d84.png)

