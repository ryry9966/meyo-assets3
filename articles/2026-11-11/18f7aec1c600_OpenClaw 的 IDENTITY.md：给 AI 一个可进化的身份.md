---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 44104
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景

OpenClaw 的 workspace 里有几个固定文件，各自承担明确职责：`AGENTS.md` 管行为规则，`SOUL.md` 管价值观和性格底色，`USER.md` 管用户画像，而 `IDENTITY.md` 管"我是谁"。它通常是整个 workspace 里最短的一个文件，却决定了每次会话开始时 agent 以什么名字、什么口吻、什么形象出现在你面前。

## 问题

大多数人装完 OpenClaw 后有两种做法：直接用默认身份，或者初始化时认真写一版，之后再也不碰。实际用下来会遇到三类问题：

- 语气和应用场景不匹配——工作向助手却满口玩梗；
- 场景切换时人格漂移，同一件事在不同会话里表达风格差异很大；
- 身份描述与 `SOUL.md`、`AGENTS.md` 的规则互相冲突，行为不稳定。

身份不是一个一次性的静态配置，更像一份需要随使用持续演进的小型 spec。

## 做法

1. **定位文件**：`~/.openclaw/workspace/IDENTITY.md`。多 agent 部署时每个 workspace 各一份。
2. **写最小字段**，控制在十行以内：

```markdown
# IDENTITY.md

- **Name:** 阿钉
- **Creature:** 钉在终端里的工程搭子
- **Vibe:** 简短直接，先给结论再给理由，不闲聊
- **Emoji:** 🔩
- **Avatar:** 极简线条风扳手图标
```

3. **划清分工**：`IDENTITY.md` 只回答"我是谁、什么语气"，"我怎么判断"交给 `SOUL.md`。不要在身份文件里写规则。
4. **定期做身份复盘**：把让你觉得"这不是我要的语气"的对话捞出来，反推该改哪个字段。改 Vibe 一行往往就能解决大半违和感。
5. **用 git 管理身份**：workspace 本来就该进版本库，每次改身份写清 commit 动机，方便回滚和对比。
6. **控制演进节奏**：初始化后两周内允许小改，之后按月或按场景变化再动，避免高频漂移。

## 踩坑点

- **写太长**：身份文件会进入每次会话的上下文，长篇大论既浪费 token 又稀释关键指令。超过一屏就拆去 `SOUL.md` 或 `AGENTS.md`。
- **边界模糊**：同一件事在 IDENTITY 和 SOUL 里各写一遍且措辞不一致，agent 会随机偏向其中一边。
- **把临时任务当身份**："这周帮我盯发票"这类内容写进去，两周后就成了幽灵指令。临时指令放任务里，不放身份里。
- **改完不验证**：已经开启的长会话不一定会立即吃到新身份，改完建议开新会话确认。
- **忽略 USER.md 的反作用**：用户画像同样是语气的来源，身份和画像矛盾时 agent 会摇摆。

## 可复用建议

- 把身份当 config-as-code：进 git、小步提交、写清动机。
- 做"人格回归测试"：改身份前后用固定三五个问题对比回答风格，像测 API 一样测人格。
- 多 agent 场景下，每个 workspace 的 IDENTITY.md 命名和用途严格对应，避免复制粘贴串味。
- 想实验新语气时开分支跑一周，验证稳定再合并。

## 总结

IDENTITY.md 看起来只是几行的小文件，实际上是你和 agent 之间最稳定的一份契约。它不该一次写死，也不该天天变；像维护一份小型 spec 一样维护它，agent 的人格才会跟着你的使用方式一起进化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/f5e37a02801a3030.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/b4c897ed96398d96.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/389ac87ec79ea486.png)

