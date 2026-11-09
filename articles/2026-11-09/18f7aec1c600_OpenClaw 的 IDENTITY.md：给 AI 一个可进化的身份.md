---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 43929
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

OpenClaw 的 Agent 是长期运行的：跨会话、跨渠道、跨模型。但 LLM 本身无状态，每次新会话，模型对"我是谁"一无所知。OpenClaw 的做法是把人格外置到工作区文件里——SOUL.md 管行为边界，USER.md 记用户事实，而 IDENTITY.md 专门回答一个问题：这个 Agent 是谁。

## 问题

早期把身份写死在 system prompt 配置里，有三个实际痛点：

1. 改一次人格要动配置，迭代成本高；
2. 没有历史，改坏了不知道之前长什么样；
3. 多 Agent 场景下，几个实例人格容易"串味"，路由到哪个渠道都像同一个人。

## 做法

IDENTITY.md 位于 Agent 工作区（默认 `~/.openclaw/workspace/`，多 Agent 时在各自 workspace 下），每次会话装配 system prompt 时注入。典型内容很短：

```markdown
# IDENTITY
- Name: Nova
- Creature: 猫头鹰
- Emoji: 🦉
- Vibe: 克制、工程化、直给结论
- Avatar: avatars/nova.png
```

让它"可进化"，建议这样跑：

1. **纳入版本控制**：把整个 workspace 放进 git，身份的每次修改都是一个可回滚的 diff；
2. **提案制**：想调整人格时，先让 Agent 起草 `IDENTITY.draft.md`（prompt 里明确"只提案，不落盘"），人工 review 后合并；
3. **低频反思**：用 heartbeat/定时任务每月跑一次，让 Agent 基于近期记忆提出身份修订建议，而不是每轮对话自我修改；
4. **新会话验证**：改完开新会话确认生效——运行中的旧会话仍持有旧身份。

## 踩坑点

- **职责混装**：把"回答要带出处""别执行危险命令"这类规则塞进 IDENTITY.md，文件越写越长。规则归 SOUL.md，事实归 USER.md，身份只描述"是谁"。
- **让 Agent 直接自改**：模型倾向于把自己改得更迎合，几轮下来人格漂移明显，务必走 draft + review。
- **误判不生效**：长会话不会热加载身份，别误以为是没写进去。
- **改错文件**：多 Agent 时编辑了默认 workspace 的文件，目标 Agent 读的却是 `agents/<id>/workspace` 下那份。
- **写太长**：超过二三十行，注入成本上升且关键特征被稀释，三句话能说清的 vibe 就别写三段。

## 可复用建议

- **身份即版本化资产**：git diff 就是人格变更日志，出问题一键回滚；
- **模板 + 覆盖**：维护一份通用模板，各 Agent 只 override Name/Creature/Vibe 三行，多实例人设一致性可控；
- **低频演进优于高频 churn**：身份应像域名一样稳定，而不是像话题一样天天变；
- **emoji 和 avatar 别留空**：它们出现在启动消息和部分渠道展示里，是身份的一部分，不只是装饰。

## 总结

IDENTITY.md 的价值不在文件本身，而在于它把"AI 是谁"从一段 prompt 片段，变成了可版本化、可评审、可回滚的工程资产。一个小文件，配上一条 draft → review → merge 的演进流程，就能让 Agent 在长期运行中保持人格稳定，同时留出合理的进化空间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/4863c1456f512c91.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/4a0ebb0e663f68df.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/a2585e8e4959fef8.png)

