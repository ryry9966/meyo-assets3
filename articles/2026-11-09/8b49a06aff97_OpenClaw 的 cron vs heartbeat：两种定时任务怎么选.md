---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 43870
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

OpenClaw 里有两套让 agent“定时醒来”的机制。**cron** 是经典调度：标准 5 段 cron 表达式 + 时区，到点就跑，默认跑在独立会话里；**heartbeat** 是间隔心跳：默认每 30 分钟在主会话里唤醒一次 agent，由它自己判断“要不要做事”，没事就回一个 `HEARTBEAT_OK` 静默通过。

机制本身都不复杂，问题出在选型。新手常见两种极端：任务全塞给 cron，任务表越拉越长；或者全靠 heartbeat，时间敏感的事总是差半小时才发生。

## 问题

两者差别的本质是**控制权归属**：

- cron：你决定它什么时候醒，醒来干什么由你写好的 payload 决定；
- heartbeat：它按固定节奏醒，醒来干什么由 agent 现场判断。

选错机制会带来三类麻烦：时间敏感任务受心跳间隔误差影响；观察类任务被建成一堆 cron，频率高了烧 token、低了失去监控意义；heartbeat prompt 写得太开放，agent 每次醒来都“找点事做”，账单和噪音一起涨。

## 做法

1. **先分类任务**。有明确时间锚点的（每天 9:00 发日报、周五下午汇总周报）→ cron；只是“定期看一眼”（收件箱有没有要紧消息、某个服务状态是否正常）→ heartbeat。
2. **配 cron 时三件事写全**：schedule 用标准 5 段表达式并显式指定 tz；payload 必须自包含——独立会话里没有历史上下文，数据来源、输出渠道、失败兜底都要写进这个 prompt。
3. **配 heartbeat 时两件事收敛**：用 `heartbeat.every` 把间隔调到业务能接受的粒度；在心跳 prompt 里写清静默规则——“无事则不输出、不动手”，有明确阈值的（如错误率超过 X% 才通知）写成硬条件。
4. **允许混合**。heartbeat 当哨兵，发现异常后再由 agent 通过 cron 工具补一条精确任务，或你手动升级。哨兵负责发现，cron 负责执行。

## 踩坑点

- **时区**。VPS 多为 UTC，cron 表达式忘配 tz，日报在下午才发。这是最常见的一类事故。
- **状态依赖**。在 cron payload 里写“接着上次的进度”，但独立会话里没有“上次”。中间状态必须落盘（文件或外部存储），谁醒来都能读。
- **心跳太密**。间隔小于典型任务耗时会出现唤醒排队叠加，token 消耗先涨，日志噪音随后涨。
- **用 heartbeat 做“必须在整点发生”的事**。心跳本身有抖动，误差可能到分钟级，别指望它踩点。

## 可复用建议

- 一句话决策：**时间点明确 → cron；状态观察 → heartbeat**。
- cron prompt 四要素模板：目标、数据来源、输出渠道、失败怎么办，四项缺一不可。
- 每月审计一次心跳日志：每次醒来都有事的 heartbeat，升级成 cron；长期无事的发生降频或删掉。
- 所有定时任务的中间结果写文件，不依赖会话记忆，cron 和 heartbeat 才能互相接力。

## 总结

cron 和 heartbeat 不是竞争关系，是分工：**cron 管“日程”，heartbeat 管“巡逻”**。把精确的事交给 cron，把模糊的事交给 heartbeat，agent 的自主性才有边界，你的 token 账单和运行日志也才看得懂。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/e7d23e0c4d27504e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/d7d60fb9f8443d91.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/46b6fb18bdec5304.png)

