---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 43884
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

OpenClaw 这类常驻 agent 的价值，很大程度在"不需要你盯着"。它内置了两种定时触发机制：**cron**（类 Unix crontab 的定时作业）和 **heartbeat**（固定间隔的心跳巡查）。两者看起来都是"到点干活"，但设计意图完全不同，混用是社区里最常见的浪费和翻车来源之一。

## 问题

实际使用中经常出现两类极端：

- **全用 cron**：每 15 分钟挂一个"帮我看看有没有新东西"的作业，会话数爆炸，token 账单难看，结果还散落在各个孤立会话里。
- **全用 heartbeat**：把"每天 8 点发晨报"写进 HEARTBEAT.md，靠 agent 自己判断"现在几点"——轻则迟到，重则干脆跳过。

## 两种机制的核心差异

**cron：时间驱动，隔离会话。** 每个作业在独立 session 里跑，用 cron 表达式精确到分钟，跑完可把结果投递到指定渠道（Telegram/WhatsApp 等），适合一次性提醒、每日摘要这类"几点做什么"的任务：

```bash
openclaw cron add --name morning-digest \
  --cron "0 8 * * *" \
  --message "汇总昨天未读消息和今日日程，生成晨报并发出去"
```

**heartbeat：状态驱动，共享主会话。** gateway 按固定间隔（默认 30 分钟）在主会话里喂给 agent 一次心跳 prompt，agent 检查工作区的 HEARTBEAT.md 清单：没事就回 `HEARTBEAT_OK`（该回复会被吞掉，不打扰你），有事才行动。适合"帮忙盯着点""顺手清理一下"这类轻量巡查，且因为共享主会话，能延续上下文。

## 选型步骤

1. **先问触发条件是时间还是状态**："8 点整发" → cron；"每隔一段时间看看有没有要处理的" → heartbeat。
2. **cron 检查三件事**：表达式的时区（以 gateway 主机为准）、投递目标是否配置、是否设了单次超时。
3. **heartbeat 做减法**：HEARTBEAT.md 保持 3~5 条以内的小清单，任务写成幂等操作；暂时用不到就把间隔拉长或直接关掉，别让它空转烧 token。
4. **组合使用**：固定时间的重活交给 cron，间歇性轻巡交给 heartbeat，各司其职。

## 踩坑点

- **时区**：VPS 上 gateway 多为 UTC，`0 8 * * *` 实际是北京时间 16 点。先加一个测试作业验证投递时间，再上正式任务。
- **heartbeat 烧 token**：即使清单为空，每次心跳也要过一遍模型。间隔低于 10 分钟基本是烧钱行为。
- **重活别放 heartbeat**：心跳在主会话里跑，长任务会阻塞正常对话，还会把上下文撑爆。
- **cron 无补跑语义**：gateway 重启期间错过的任务通常不会自动补，关键任务要加事后核对，别假设"到点必达"。
- **HEARTBEAT.md 腐化**：已废弃的旧任务残留在清单里，agent 会反复执行。改完清单让它跑一轮心跳确认结果。

## 可复用建议

一句话决策：**时间驱动用 cron，状态驱动用 heartbeat**。再补三条：

- cron 作业全部命名并写清用途，一个月后你会感谢自己；
- HEARTBEAT.md 当"值班巡检表"而不是任务队列，大任务拆给 cron 或手动触发；
- 每周看一眼 cron 会话列表和 token 用量，把低价值作业删掉。

## 总结

cron 和 heartbeat 不是竞争关系，而是两种触发语义：前者是"闹钟"，精确但隔离；后者是"巡逻"，模糊但连续。把"几点必须做"交给 cron，把"没事就别吵我"交给 heartbeat，OpenClaw 的自动化才不会变成 token 无底洞。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/61e5270dea0c9cdd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/ef6f4a13f9a0d506.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/877f411a566c4a16.png)

