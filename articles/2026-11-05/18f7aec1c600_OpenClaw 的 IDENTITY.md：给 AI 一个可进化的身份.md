---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 43488
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

OpenClaw 的工作区里有一组 Markdown 文件充当 agent 的常驻配置：`AGENTS.md` 管行为规则，`SOUL.md` 管价值观和做事方式，`IDENTITY.md` 管"我是谁"。多数教程把篇幅给了 SOUL.md，IDENTITY.md 往往是启动向导自动生成一次之后就再也没被打开过。这篇讲的是怎么把它当成一份真实的配置资产管理起来。

IDENTITY.md 的典型字段是 Name、Creature、Emoji、Vibe、Theme 这几项，会在会话构建时注入 system prompt，模型据此维持自我描述的一致性。它看起来像装饰，实际上是每轮都在生效的上下文。

## 问题

没有稳定的身份锚点，实践中会撞到三类问题：

1. **行为漂移**。长会话或者切换模型版本后，agent 的自称、语气、汇报风格不稳定，排查起来像玄学。
2. **多 agent 混淆**。同时跑工作、家庭、测试三四个 agent，如果它们共享一套模糊的"我是助手"认知，写入 MEMORY.md 的内容和主动汇报时经常分不清是谁的。
3. **演化无据可查**。调过几次人格，但说不清什么时候改的、为什么改，想回滚只能凭记忆。

## 做法

1. **先做减法**。把向导生成的默认文件读一遍，删掉空话，只保留能实际约束行为的字段。
2. **字段最小化**。Name / Creature / Vibe / Theme 够用，再加一行 Boundaries（不自称什么、不碰什么话题）——一行硬约束比十行形容词有用。
3. **版本化**。workspace 整体纳入 git，每次身份调整一个 commit，message 写触发原因，比如"汇报口吻太轻佻，收紧"。
4. **绑定复盘节奏**。身份演化跟着 MEMORY.md 的周期复盘走：每隔一两周翻会话记录，把反复出现的语气问题收敛成一次字段修改，而不是发现一次改一次。
5. **生效验证**。改完开新会话，用固定的三个问题问它：你是谁、你怎么称呼自己、遇到不确定的事怎么办。把回答和上一版对比，三个问题可以直接写进一个 fixtures 文件复用。

## 踩坑点

- **写太长**。IDENTITY.md 是每轮注入的高价值上下文，超过一屏就是在烧 token，形容词堆叠会稀释关键约束。
- **和 SOUL.md 抢活**。IDENTITY 回答"我是谁"，SOUL 回答"我怎么做事"，同一件事写两处，迟早改一处忘另一处。
- **改了不生效**。它在新会话构建时加载，复用的旧 session 不会自动刷新，验证前先确认开的是新会话。
- **多 agent 路径**。`workspace-<agent>` 各自独立，主 agent 的改动不会传染给子 agent，别替它们做主，各改各的。
- **别忽视 Emoji 和 Theme**。看着是装饰，实际影响通知渠道里的辨识度和语气暗示，认真选。

## 可复用建议

- 字段收敛到五行以内：Name / Creature / Vibe / Theme / Boundaries。
- "固定三问"做成回归用例，每次换模型版本跑一遍，对比输出。
- 一次只改一个字段，观察几天再改下一个——身份也适合小步提交。
- 每次修改的动机写进 commit message，半年后你会感谢现在的自己。

## 总结

IDENTITY.md 的价值不在于"给 AI 起个名字"，而在于把行为一致性里最容易玄学化的部分，落成一个可 diff、可回滚、可评审的文件。它不会让 agent 更聪明，但能让它在长周期里保持是你配置出来的那个它。从工程角度看，这比任何提示词技巧都便宜，也更容易积累。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/a3f112ab2b7c055f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/5c6b3ae1d82884ab.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/9c205380802dbea5.png)

