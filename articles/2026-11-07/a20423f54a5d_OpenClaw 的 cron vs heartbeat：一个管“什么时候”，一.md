---
title: OpenClaw 的 cron vs heartbeat：一个管“什么时候”，一个管“要不要”
feedId: 43749
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

OpenClaw 的 agent 是常驻进程，很多人装完第一件事就是让它“定时干活”。框架里其实有两条独立的定时路径：

- **cron jobs**：用 cron 表达式精确触发，到点把一段你写好的固定 prompt 交给 agent，时间和动作都是确定的。
- **heartbeat**：按固定间隔（默认 30 分钟）唤醒 agent，让它读一遍 HEARTBEAT.md 里的检查清单，自己判断“现在有没有事需要做”。

一个是闹钟，一个是巡逻。机制不同，适合的任务也不同。

## 问题

新手最常见的两类翻车：把“每天早 9 点发日报”写进 HEARTBEAT.md，指望模型到点自觉执行，结果忽早忽晚甚至漏掉；反过来用一串 cron 表达式模拟“帮我盯着服务器，有异常再说”，每小时硬触发一次全量检查，token 账单很难看。

本质是没回答一个前置问题：**这个任务的触发时间是确定的，还是需要 agent 判断的？**

## 做法

1. **先分类任务。** 固定时间 + 固定动作（日报、提醒、备份触发）→ cron；条件触发 + 容忍延迟（异常巡检、磁盘水位关注、收件箱清理建议）→ heartbeat。
2. **配置 cron。** 用 `openclaw cron add` 写清表达式和 prompt。注意 session 隔离策略——隔离 session 里 agent 看不到主会话记忆，prompt 必须自包含。
3. **配置 heartbeat。** 把条件型任务写成 HEARTBEAT.md 清单，用 heartbeat.prompt 约束行为：无事可做就静默跳过，有事才输出。间隔先保持默认 30 分钟。
4. **时间精确但动作依赖状态时，用混合模式：** cron 负责到点叫醒，prompt 里只写“检查 X 条件，满足再做 Y”，判断权留给 agent。

## 踩坑点

- **heartbeat 间隔调太短。** 每次心跳都是一次完整的 LLM 调用，从 30 分钟改到 5 分钟，成本约涨 6 倍——哪怕 90% 的心跳什么都没干。
- **HEARTBEAT.md 当定时器用。** “到点必须做”的事模型会漏、会拖，这类任务一律下沉到 cron。
- **cron 时区。** 容器里默认 UTC，`0 9 * * *` 实际对应北京时间下午 5 点，部署时显式指定时区。
- **两类触发撞车。** cron 和 heartbeat 同一时刻醒来，输出容易交错混乱，跑几天日志再排布触发时间。
- **心跳静默 ≠ 没执行。** agent 判断“无事”时不输出，容易被误认为心跳挂了。留一个低频的“心跳自检”任务验证链路通不通。

## 可复用建议

一句话原则：**cron 管“什么时候”，heartbeat 管“要不要”。**

- HEARTBEAT.md 控制在 5 条以内，只放条件型、可延迟任务；定期删掉长期没命中的条目。
- 心跳参数先观察一周再调；强模型 + 短清单，通常比弱模型 + 长清单更省也更稳。
- cron 的 prompt 一律写成无上下文也能执行的自包含格式，别依赖“它应该记得”。
- 定期 `openclaw cron list` 审计，清掉一次性任务（比如临时提醒）和过期的报告推送。

## 总结

两种机制不是二选一，而是分工：时间敏感、结果确定的活交给 cron；需要“看情况”的值守交给 heartbeat。混用的代价不是功能失效，而是不准时、漏执行和看不见的 token 消耗。先分类，再落地，两套机制搭在一起才稳。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/70f33ce06fe38b57.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/aec51a27fed74155.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/8cdd54cb0c7ac730.png)

