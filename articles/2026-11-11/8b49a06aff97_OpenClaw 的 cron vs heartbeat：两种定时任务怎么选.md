---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 44045
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景

在 OpenClaw 里做自动化，内置两条路：

- **cron**：确定性定时器。到点触发一次任务，prompt 固定，结果可预期。
- **heartbeat**：心跳机制。按固定间隔给 agent 注入一条心跳 prompt，agent 读自己的检查清单（如 `HEARTBEAT.md`），自行判断这一轮是干活还是跳过。

两者的本质区别不在"定时"，而在**谁来决策**：cron 由调度器决策，agent 只负责执行；heartbeat 把决策权交给 agent。

## 问题

社区里常见的两种错配：

1. 把 heartbeat 当 cron 用——"每天 9 点发日报"写进心跳清单。结果是时间漂移、token 白烧，还占用心跳轮次。
2. 把需要判断的巡检拆成十几个碎片 cron——"每小时查邮箱、每半小时看 PR、每天整理笔记"。任务不会自己收敛，改一次要动一堆 job。

## 做法：一条选型规则

**时间是死的用 cron，判断是活的用 heartbeat。**

cron 适合：

- 时间点明确、动作固定的任务：定时日报、每日摘要推送、周期性备份。
- 任务描述要自包含：cron 触发的是一次相对干净的会话，别指望它记得"上次聊到哪"。

heartbeat 适合：

- 何时该做不确定的事：巡查邮箱 / PR / 消息、"有空就整理素材"。
- 依赖会话上下文与记忆的连续性任务。

两者可以组合：heartbeat 巡检中发现"需要在精确时间做的事"，就动态注册一个一次性 cron，把判断的结果交给确定性的执行。

配置要点（以你当前版本文档为准）：cron 通过 `openclaw cron` 子命令或配置注册，注意显式指定时区；heartbeat 在配置里调间隔，清单文件保持短小、可判定。

## 踩坑点

1. **心跳有成本**：每次心跳都是一次真实的 LLM 调用，哪怕 agent 决定跳过也花钱。任务密度低就把间隔调大。
2. **心跳 prompt 要收敛**：明确约定"无事可做就回复固定短语"，否则它每轮都可能长篇输出，日志没法过滤。
3. **时区**：容器里常是 UTC，`0 9 * * *` 实际是北京时间 17 点。不显式指定时区的 cron 迟早翻车。
4. **重叠执行**：长任务 + 短心跳会导致并发堆积，加并发锁，或保证任务幂等。
5. **别拿 heartbeat 做精确调度**：它的精度是"分钟级 + LLM 判断"，到点必达的需求一律 cron。

## 可复用建议

- 巡检类需求合并成**一张** heartbeat 清单，而不是 N 个 cron；清单条目写成"能一句话判定做/不做"。
- cron 的任务 prompt 按"丢给一个没有任何上下文的新 agent 也能执行"的标准写。
- 日志里按心跳的固定跳过短语做过滤，方便估算心跳的真实开销。
- 新想法先跑 heartbeat 原型验证有没有价值，模式稳定后再 cron 化固定时间，避免过早固化。

## 总结

cron 和 heartbeat 不是替代关系，而是分工：**cron 负责确定性，heartbeat 负责自主性**。动手前先问一个问题——"这个任务需要 agent 判断吗？"不需要，就交给 cron；需要，就写好清单交给心跳，并管好间隔与成本。选型对了，自动化才省心。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/7765c661ca0f2b2b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/ac794a0e5cf326c3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/a3ec3d846dbddbc6.png)

