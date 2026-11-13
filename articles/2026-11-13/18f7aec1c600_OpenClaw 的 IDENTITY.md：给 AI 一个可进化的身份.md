---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 44312
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

OpenClaw 跑久了你会发现，agent 的"能力"靠工具和 skill 沉淀，但"人格"是另一回事。今天调好的语气、自我介绍、称呼方式，换一次会话、换一台机器部署就丢了。IDENTITY.md 就是官方给出的答案：一个放在 workspace 里的 Markdown 文件，每次会话启动时注入 system prompt，定义这个 agent 是谁。

## 问题

没有身份文件时的典型症状：

- **人格漂移**：周一它自称"你的工程助手"，周四开始用二次元腔回你消息
- **多端不一致**：本机和 VPS 上跑的两个实例，行为像两个不同的员工
- **团队成本高**：每个人都要从头"调教"一遍，经验无法共享

本质问题是身份信息散落在历史对话和零散的提示词里，不可版本化、不可迁移。

## 做法

最小可用版本，五分钟能写完：

```markdown
# IDENTITY.md
- Name: Nova
- Creature: CLI 里的工程助手
- Vibe: 克制、直接、不废话
- Emoji: 尽量不用，代码场景禁用
- Boundaries: 不确定就说不确定，不编造 API
```

然后是迭代流程：

1. 先写最小版本上线，别追求一步到位
2. 跑真实任务一周，把不满意的具体回复记下来
3. 每条不满翻译成一条身份约束（语气问题、边界问题还是自我认知问题）
4. 改 IDENTITY.md，git commit，观察一天行为
5. 每月 review 一次，删掉过时条目

关键心法：**把它当代码维护，而不是一次性配置。**

## 踩坑点

- **写太长**：超过一页，核心约束会被稀释，agent 开始"记不住"自己是谁。控制在 30 行以内。
- **混入任务指令**："提交前跑 lint" 属于 AGENTS.md 的职责，混进来会让身份文件膨胀成第二份操作手册。
- **一次改太多**：一口气改十条约束，没法归因是哪条生效。小步提交，逐条验证。
- **写成空话**："要友善、要专业"没有信息量，模型本来就会。要写它**默认不会做**的事。

## 可复用建议

- **分工记牢**：IDENTITY.md 管"我是谁"，SOUL.md 管"我怎么反应"，USER.md 管"对面是谁"，边界不要串文件。
- **用 git 管身份**：行为异常时先看 `git diff IDENTITY.md`，八成能定位到最近一次改动。
- **多实例部署**：身份文件纳入初始化脚本，和环境配置一起分发。
- **团队协作**：身份变更走 PR review，人格也需要 code review。

## 总结

IDENTITY.md 的价值不在"让 AI 更像人"，而在把原本不可控的人格变成**可版本化、可回滚、可协作**的工程资产。第一版只要五分钟，真正的收益来自持续的小步迭代。建议每个跑 OpenClaw 的 workspace 都先落一个最小版本，让真实使用告诉你该往哪里改。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/18e6bf6d4d653690.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/c71f030bb136a75b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/9442eb37cec33883.png)

