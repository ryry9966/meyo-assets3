---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 44327
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

OpenClaw 的 agent 不是“一坨 prompt”，而是围绕 workspace 里一组 Markdown 文件运转的：`AGENTS.md` 管协作规则，`SOUL.md` 管行为基调，`USER.md` 管对用户的了解，而 `IDENTITY.md` 管的是最基础的问题——“我是谁”。默认模板围绕几个字段展开：名字、物种/形态、emoji、avatar，外加一小段自我描述。它会在每次 session 启动时被拼进系统提示，所以这不是一个摆设文件，而是每个会话都会生效的“人格源文件”。

## 问题

没有这份文件（或者随便填了事）时，常见症状有三个：一是模型默认人格直接漏出来，换模型、升版本之后语气漂移；二是多渠道场景（群聊、私信、定时任务）里同一个 agent 表现不一致；三是同时跑多个 agent 时，日志和群聊里分不清谁是谁，用户只能靠猜。更麻烦的是，如果人格散落在代码里的 system prompt 和散装配置中，每次调整都要动代码、重启，谈不上“进化”。

## 做法

1. **先找到文件**：agent workspace 下的 `IDENTITY.md`，从默认模板起步，别从零发明格式。
2. **最小可行身份**：只填 name + 一句性格描述 + emoji/avatar，控制在十行以内。它会进每次会话的上下文，写长了就是纯烧 token。
3. **职责分离**：身份文件只放“稳定特质”——名字、自称、语气基线、边界感；具体行为规则放 `SOUL.md`/`AGENTS.md`，用户相关信息放 `USER.md`。混着写迟早打架。
4. **让它可进化**：把 workspace 纳入 git。实践中比较稳的循环是：会话中发现语气不对 → 让 agent 提一份 IDENTITY.md 修改提案 → 人审 diff → commit。身份变更从此有历史、可回滚。
5. **多 agent 场景**：每个 agent 有独立 workspace，刻意拉开 name/emoji/avatar 的区分度，群聊和日志里一眼可辨。

## 踩坑点

- **写太长**：有人把大段人设全塞进去，结果任务指令被稀释，回答反而变差。身份文件贵精不贵多。
- **期望热更新**：文件在 session 启动时读取，会话中途改了不生效，新会话才加载。调试时别被这个坑骗。
- **让 agent 自改不审查**：模型倾向往“更讨好”的方向改自己，几次迭代人格就漂了。diff 必须过人眼。
- **塞隐私**：这份文件每个会话都带上，用户敏感信息别放这里，放 `USER.md`/`MEMORY.md` 并按需控制。
- **模型升级后不复检**：上游模型默认风格变了，旧身份描述可能压不住，升级后跑几个典型场景对照一下。

## 可复用建议

- **把身份当代码管**：改身份走分支和 commit message（如 `identity: 错误场景收敛语气`），别直接在运行环境里手改。
- **团队共用时沉淀字段模板**：name / role / tone / boundaries / escalation，新人接手只改这五项。
- **每月一次 diff 回顾**：把过时的描述清掉——身份文件也怕腐化。

## 总结

IDENTITY.md 的价值不在文件本身，而在于它把“agent 是谁”从一个藏在代码里的常量，变成了一个有版本、有审查、有演进记录的资产。写短、管好职责边界、改动走 review，身份就会随使用越磨越准，而不是越养越歪。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/5c292283d7ec2354.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/c7bff02bd3e0b809.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/119c70584ff676f8.png)

