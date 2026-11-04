---
title: IDENTITY.md 实践：给 OpenClaw 一个可进化的身份
feedId: 43486
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

OpenClaw 的工作区里有几个约定文件：`AGENTS.md` 管行为规则，`SOUL.md` 管价值倾向，`TOOLS.md` 管工具备忘，`USER.md` 管对用户的记忆。其中最容易被晾着的是 `IDENTITY.md`——它定义 agent 的名字、形象、语气、emoji、头像，并在每次会话启动时注入上下文。大多数人装完就留着默认模板，跑了几个月回头一看，助手还是那个"出厂人设"。

## 问题

我在多 agent 场景下踩到的实际痛点：

1. **人设散落**。早期我把语气要求直接写进 `AGENTS.md`，规则和人设搅在一起，改个语气要连带动规则，diff 一团糟。
2. **跨会话漂移**。身份没落盘，模型每次自拟，今天高冷明天话痨。
3. **多助手串味**。工作助手和家里助手共用一套 workspace，人格互相污染，通知里甚至@错名字。
4. **无法追溯**。想不起来"上次为什么把语气改成简洁模式"。

根因只有一个：身份没有被当成一份需要版本管理的配置。

## 做法

1. **单独成文**。把人设从 `AGENTS.md` 全部迁出，`IDENTITY.md` 只放身份层：name、creature、vibe、emoji、avatar，外加两节自定义内容——「语气示例」（给一段正例、一段反例）和「边界」（不自称什么、不摆什么姿态）。规则和价值继续留在 `AGENTS.md` / `SOUL.md`。
2. **进 git**。整个 workspace 纳入私有仓库，身份变更单独提交，commit message 用 `identity:` 前缀，方便 `git log -S` 追溯每一次"换人格"。
3. **设计进化回路**。这是"可进化"的关键：每次遇到语气不对的对话，往 workspace 里一个 `friction.md` 追加一行记录；每月让 agent 基于这些记录起草一版 `IDENTITY.md` 的 diff，我人工审完再合并。改身份的永远是人，agent 只有起草权。
4. **多实例隔离**。给不同 agent 各自的 workspace，`IDENTITY.md` 里明确写"你不是谁"，实测对防串味有效。

## 踩坑点

- **文件别写长**。`IDENTITY.md` 每次会话都注入，我第一版写了 200 行，token 白烧；压缩到 50 行内，没感知到任何效果损失。
- **emoji 控制在 1–3 个**。部分通知渠道会拿它当消息前缀，塞一堆很难看。
- **别让 agent 无审批自改**。放开过一次，两周后人设的讨好倾向明显加重——自我修改会放大它自己偏好的表达方式。
- **大改名留版本**。文件头维护一行 version 注释，重大改版记一句动机，半年后回看能救命。

## 可复用建议

- 把 `IDENTITY.md` 当产品设计文档维护：月度小修，季度大审，节奏低频但固定。
- 语气示例比形容词有用。"回复要简洁"不如放一段 50 字以内的正例。
- 心智模型：`AGENTS.md` 是"法律"，`SOUL.md` 是"价值观"，`IDENTITY.md` 是"名片"。分层清晰，改动才安全。

## 总结

`IDENTITY.md` 的价值不在那几行元数据，而在把身份从"随缘生成"变成一份**有版本、有回路、有人守门的资产**。花一个下午迁好、进 git、建好 friction 记录，你的 agent 才算真正拥有一个能长期对齐、也能慢慢长大的人格。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/ee291658f9f7c5c7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/8359b898536009bf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/2472a9d0e71a0fd9.png)

