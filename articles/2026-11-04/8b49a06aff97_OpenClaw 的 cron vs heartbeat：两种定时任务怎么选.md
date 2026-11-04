---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 43446
source: 综合讨论
publishedAt: 2026-11-04
---

## 背景

OpenClaw 里有两套“定时”机制：cron 和 heartbeat。社区里常见两种极端用法：要么只用 heartbeat，每天被定时消息轰炸；要么把所有巡检堆成几十条 cron，token 账单很难看。这篇帖基于我在自己网关上的实践，梳理两者的定位差异和选型思路。

## 问题

两者都会定期唤醒 agent，但机制完全不同：

- **cron**：确定性调度。每个 job 自带独立 prompt 和排期，到点必跑，结果可投递到指定频道。
- **heartbeat**：周期性心跳（默认 30 分钟，可调）。agent 被唤醒后读取工作区的 `HEARTBEAT.md`，自行判断有没有事可做，没事就保持沉默。

核心问题只有一个：什么时候该建 cron job，什么时候该往 `HEARTBEAT.md` 里加一条？

## 做法

我的选型原则一句话：**有确定的钟点 → cron；只有模糊的状态 → heartbeat。**

### 走 cron 的情况与步骤

凡是“每天 9:00 汇总”“每周一收周报”这类有明确时间承诺的任务：

1. 用 `openclaw cron add` 创建 job，写清 cron 表达式，并**显式指定时区**，不要依赖服务器默认时区。
2. prompt 写成完整指令：做什么、输出格式、异常时怎么办。它是无人值守任务，不是对话，别指望 agent 自己补上下文。
3. 配置投递策略：常规报告按默认推送；监控类任务改成只在异常时通知，避免刷屏。
4. 用 `openclaw cron list` 和 runs 记录确认首次执行结果，再放手让它进正式排期。

### 走 heartbeat 的情况与步骤

凡是“看看有没有事”“保持环境健康”这类说不清具体时刻的巡检：

1. 把 `HEARTBEAT.md` 当成“值得醒来的条件清单”，不是任务清单。每条写触发条件加期望动作，例如“发现未处理的关键邮件 → 摘要后提醒”。
2. 文件开头写明沉默条款：没有命中任何条件时，不产出任何消息。
3. 间隔从默认 30 分钟起步，观察一周 token 消耗后再调。

两者可以共存：时间承诺给 cron，`HEARTBEAT.md` 只留 3~5 条兜底巡检。

## 踩坑点

- **heartbeat 刷屏**：`HEARTBEAT.md` 里没写沉默条款，agent 每半小时汇报一次，频道直接被淹没。反过来空文件也不好，等于每拍白烧 token。
- **间隔调太低**：有人把 heartbeat 调到几分钟一次来“模拟实时”，先崩的是 token 额度和模型限流。
- **cron prompt 太模糊**：“帮我看看邮件”这种 prompt 在凌晨三点执行，agent 的发挥不可控。
- **时区**：cron 表达式默认跟随网关所在机器，跨时区几乎必踩，逐个 job 显式指定。
- **心跳是单队列**：某次巡检触发了一个长任务，下一拍会顺延，所以别用 heartbeat 做时效敏感的事。

## 可复用建议

- 判断口诀：**钟点 → cron，状态 → heartbeat。**
- cron job 三要素：精确时间（含时区）+ 完整 prompt + 明确投递目标。
- `HEARTBEAT.md` 三要素：沉默条款 + 条件化条目 + 控制在 5 条以内。
- 新 cron job 上线前先手动触发一次，确认 runs 日志无误再进排期。
- 每月审计一次：`HEARTBEAT.md` 里条件明确到能写死的，迁去 cron；cron 里纯巡检的，迁回 heartbeat。

## 总结

cron 和 heartbeat 不是二选一，而是“确定性调度”和“自主巡检”两个象限。把时间承诺交给 cron，把模糊的“有事叫我”交给 heartbeat，再守住 heartbeat 的沉默条款和 cron 的 prompt 质量，定时任务才能做到既准时、又不吵人。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/2367bf6fb1a8dea5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/b43c2eafde20c897.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/8776d839e411ed97.png)

