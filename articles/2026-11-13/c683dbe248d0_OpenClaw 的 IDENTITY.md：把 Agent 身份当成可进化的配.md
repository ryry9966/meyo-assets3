---
title: OpenClaw 的 IDENTITY.md：把 Agent 身份当成可进化的配置来管
feedId: 44253
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

OpenClaw 的 workspace 里有一组"软配置"：AGENTS.md 定规则，SOUL.md 定行为底色，USER.md 记用户偏好，IDENTITY.md 则回答最基础的问题——这个 agent 是谁。不少教程把它一笔带过：首次启动引导你填个名字、选个 emoji，然后就再也没动过。用了一段时间后我发现，它其实是全套文件里被低估的一个。

## 问题

实际跑起来，身份会从三个地方各自长出来：system prompt 里的硬编码描述、SOUL.md 的语气设定、以及长期记忆里累积的旧称呼。后果很典型：

- 在 Telegram 里自称 A，切到另一个 channel 后又不一致；
- 改了名字，但 MEMORY.md 里的历史会话还用旧称呼，偶尔"精分"；
- 团队共用实例时，每人按自己口味改 prompt，两周后人设漂得没法看。

根因是身份没有 single source of truth。IDENTITY.md 本来就是干这个的，但它需要被当成配置资产管理，而不是一次性问卷。

## 做法

我们的实践拆成五步：

1. **写最小版**。IDENTITY.md 只放"是谁"：name、creature/emoji、vibe（两三句语气锚点）。别塞技能清单——那是 AGENTS.md 的事，塞进去只会让每轮 session 的 prompt 平白变长。
2. **纳管 git**。整个 workspace 进仓库，身份的每次修改都有 commit message 说明动机，可 diff、可回滚。
3. **定进化节奏**。约定只在月度 review 时改身份，避免"今天心情好改个名"。频率本身就是身份稳定性的一部分。
4. **agent 提案，人落盘**。在 AGENTS.md 加一条规则：agent 发现现有身份与实际场景不匹配时，把提案写进 memory；人审核后手动合入。进化方向由人把关，agent 只提供观察。
5. **更名要留痕**。改名时在 USER.md 或 MEMORY.md 里加一行"曾用名 X，某日起更名 Y"，给历史会话的引用留解释路径，不必清洗全部记忆。

## 踩坑点

- **写成散文**。最初我把它写成三段自我介绍，agent 抓重点不稳定。改成键值式结构后行为一致多了——给机器读的文件，别按人读的标准写。
- **职责越界**。把"回复要简短"这类行为规则塞进 IDENTITY.md，结果 SOUL.md 调了半天不生效。身份文件和行为文件严格分离，出问题才知道该查哪。
- **多实例共享**。图省事把两个实例的 IDENTITY.md 做了软链接，改一个动两个，排查了半天"它为什么知道另一个的名字"。
- **迁移丢文件**。重装机器时只同步了 skills 和 memory，IDENTITY.md 漏了，agent 退化成无名状态，才意识到备份脚本没覆盖它。

## 可复用建议

- 保持 IDENTITY.md 在 20 行以内，超了就说明有内容该搬走；
- 身份变更走 PR review，哪怕是一个人的项目——diff 是最好的自省工具；
- bootstrap 新实例时先复制模板再改，不要从头手写；
- 备份脚本把 workspace 的全部 md 文件当一个整体对待。

## 总结

IDENTITY.md 的价值不在于"起了个好名字"，而在于把身份从散落的 prompt 片段收敛成一个可版本化、可回滚、可审计的最小文件。身份即配置，配置即代码——小步提交，定期 review，让它跟着使用场景一起长，而不是由某一次灵光一现定终身。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/f6c2b524c2af9f47.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/923676f58459c6c7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/9d8652e894b3dc37.png)

