---
title: cron vs heartbeat：OpenClaw 定时任务选型与踩坑实录
feedId: 44196
source: 综合讨论
publishedAt: 2026-11-12
---

## 背景

用 OpenClaw 做自动化一段时间后，几乎所有人都会碰到同一个问题：想让 agent「定时干活」，系统里却给了两条路——cron 和 heartbeat。文档都读了，但什么时候该用哪个，实践中经常选错。这篇把我们踩过的坑和最终沉淀下来的选型规则整理一下，供参考。

## 两者到底差在哪

**cron：时间驱动。** 写一个调度表达式（或一句自然语言），到点触发，跑一次预置 prompt，结果投递到指定 session 或渠道。触发确定，跑完即止。

**heartbeat：状态驱动。** agent 按固定间隔醒来（默认 30 分钟，可配），读取工作区里的 HEARTBEAT.md，自己判断「当前有没有值得出手的观察项」。没事就静默，不投递任何消息。

一句话概括：cron 回答「几点做什么」，heartbeat 回答「有什么事需要我现在管」。

## 怎么选：三步判断

1. **触发条件能否写死？** 能写成「每天 08:00」「每周一 09:00」这类确定时间的，一律 cron。比如早报摘要、周报汇总、定时提醒。
2. **条件只能描述、不能定时的，用 heartbeat。** 比如「盯着某个 PR 有没有新 review」「磁盘超过 80% 再提醒我」。这类任务在 HEARTBEAT.md 里写一条：检查什么 + 满足什么条件才说话。
3. **分钟级高频轮询两个都别用。** 用外部脚本做检测，事件触发后通过消息渠道把结果推进来，agent 只做兜底处理。

## 配置要点

- **cron 的 prompt 必须自包含**：目标、数据来源、输出格式、投递目标，四样写全。触发时 agent 拿到的只有这份 prompt，别指望它「记得」之前的对话。
- **HEARTBEAT.md 当观察清单用**：条目控制在 5 条以内，每条写清「检查动作 + 出声条件」。间隔用默认 30 分钟起步。

## 踩坑点

1. **heartbeat 烧 token 是持续的。** 即使每次都静默，每次醒来也是一次模型调用。有人调成 5 分钟一跳，一晚上烧掉平时几天的量。
2. **HEARTBEAT.md 塞太满会污染主 session。** 心跳跑在主 session 上，观察项里带一堆细节，日常对话质量明显下降。细节放 cron，心跳里只留一行判断规则。
3. **cron prompt 写得太虚。** 「帮我看看邮件」这类 prompt 触发后基本产出废结果。要写成「读取 X，筛选满足 Y 的条目，按 Z 格式发到 W」。
4. **时区。** cron 跟随 gateway 所在机器的时区。服务器在 UTC、你人在东八区，任务就会差 8 小时，部署前先确认机器时间。
5. **静默不等于没跑。** heartbeat 判断无事时不会投递消息，很多人误以为它没执行。排查先看 gateway 日志，而不是反复改配置。
6. **同一件事两边都配。** cron 和 heartbeat 同时盯一个目标会收到重复提醒，一个任务只归属一条路径。

## 可复用的经验

- 选型口诀：**能写成时间表的用 cron，只能写成观察者的用 heartbeat。**
- cron 主力、heartbeat 兜底：日常自动化九成走 cron，heartbeat 只留真正需要「值守」的三五件事。
- 每月 review 一次 HEARTBEAT.md，观察项完成了就删，别让清单只增不减。

## 总结

cron 和 heartbeat 不是替代关系，而是分工关系：cron 管日程，heartbeat 管值守。前者触发确定、成本可控；后者灵活但持续耗 token，效果高度依赖你把「何时该说话」写清楚。建议先把日程类任务全部迁到 cron，再回头审视哪些事真的需要 agent 持续盯着——通常剩下的不多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/c5488bfb3bc28969.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/02a41da6980ee0e3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/4e631934d72dbdfa.png)

