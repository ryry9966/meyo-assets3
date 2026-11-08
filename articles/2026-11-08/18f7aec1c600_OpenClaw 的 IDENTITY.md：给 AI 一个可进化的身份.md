---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 43794
source: 综合讨论
publishedAt: 2026-11-08
---

## 背景

OpenClaw 的 workspace 里有一组约定俗成的 Markdown：`SOUL.md` 管性格，`MEMORY.md` 管记忆，`IDENTITY.md` 管"这是谁"。每次会话启动，这些文件会被注入上下文。`IDENTITY.md` 初始化时通常就几行：名字、形象、emoji、一句 Vibe。

问题是，大多数人初始化之后就再没打开过它。

## 问题

实际用下来有三个痛点：

1. **多 agent 无法区分。** 跑了三个 agent，通知、日志、会话里全是同一个默认名字和同一个 emoji，出了问题根本分不清是谁干的。
2. **身份与行为脱节。** 用了两个月后，agent 的实际"人格"其实长在 `MEMORY.md` 的积累里，而 `IDENTITY.md` 还停在第一天的声明，两边互相矛盾。
3. **改身份要动 prompt。** 有人把身份写死在 system prompt 或网关配置里，改一次重启一次，还没有变更记录。

## 做法

**第一步：把 IDENTITY.md 当名片写，不当简历写。** 最小字段就够：

```markdown
# IDENTITY.md
- Name: 一个好辨认的名字
- Creature: 具体的形象设定
- Emoji: 🦉（会用在通知图标上）
- Avatar: assets/avatar.png
- Vibe: 三五个短语，如"直接、抗干扰、先给结论"
```

控制在 10 行以内。它每次会话都注入，长了纯浪费上下文。

**第二步：和 SOUL.md 划清边界。** 我们的分工是：`IDENTITY.md` 回答"是谁"（名字、形象、语气基调），`SOUL.md` 回答"怎么做事"（原则、边界、风格）。写重了必然漂移，改一处忘另一处。

**第三步：给它一条进化通道。** 这是关键，做法很朴素：

1. workspace 用 git 管理，`IDENTITY.md` 的每次变更都有 diff 可看；
2. 设一个周期性维护任务（cron 或每周手动），让 agent 回顾近期 `MEMORY.md`，**提议** Vibe 或形象是否需要调整——但只出 diff，不动文件；
3. 人 review 后合入，下个新会话生效。

这样身份演进是可审计的：`git log` 里能看到它从"通用助手"一步步长成现在的样子，以及哪次事件触发了哪次修改。

**第四步：多 agent 人手一份。** 每个 agent 独立 workspace、独立 `IDENTITY.md`，名字和 emoji 必须一眼可区分。我们按职责命名，通知来了一看图标就知道是谁。

## 踩坑点

- **写太长。** 把完整人设、口头禅、背景故事全塞进去，每次会话白烧几千 token。
- **让 agent 自己改自己的身份文件。** 没有人工 review 的自我修改，几轮之后容易漂成夸张或讨好型人格。agent 提议、人批准，顺序不能反。
- **改完期待立即生效。** 在跑的会话上下文里还是旧身份，新会话才稳定。验证变更记得开新会话。
- **改错文件。** `IDENTITY.md` 必须在 agent 实际挂载的 workspace 根目录。多 workspace 场景下改了备份目录那份，等于没改。
- **emoji 随手选。** 它会出现在通知和列表里，不只是装饰。选视觉区分度高的，别一排 agent 全是同色系。

## 可复用建议

- 把最小模板存成 snippet，新 agent 初始化先填名字和 emoji，Vibe 留空跑一周再补；
- `IDENTITY.md`、`SOUL.md`、`MEMORY.md` 字段不重叠，写之前先对照另外两个；
- 身份回顾可以和记忆整理合并成同一个周期任务，一次跑完；
- 团队共用 agent 时身份可共享，memory 按人隔离——身份是公共资产，记忆是私有数据。

## 总结

`IDENTITY.md` 的价值不在那几行字，而在于它把"AI 是谁"从 prompt 里的硬编码，变成了一个有版本、有 diff、可 review 的文件。配合 git 和一个周期性回顾流程，身份会跟着实际使用慢慢长，而不是停在初始化那天。

改动小，收益直接：多 agent 不再脸盲，身份演进可追溯，上下文开销几乎为零。跑了超过两周的 OpenClaw agent，都值得回头补一次这份文件。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/8e0870ba404a9f44.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/a0bc0b16696205f9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/b03d0c4185e8530b.png)

