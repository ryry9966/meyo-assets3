---
title: OpenClaw 的 cron vs heartbeat：两种定时机制怎么选
feedId: 43978
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景

OpenClaw 网关常驻运行之后，迟早会遇到"定时找 Agent 干活"的需求。框架里有两套机制可以做这件事：

- **cron**：精确的时间表触发，到点执行指定 payload；
- **heartbeat**：固定间隔唤醒，Agent 读一遍 workspace 里的 `HEARTBEAT.md`，有事处理，没事回一句 `HEARTBEAT_OK` 就结束。

两者都能周期性触发 agent session，但语义完全不同。混着用是社区里最常见的翻车来源。

## 问题

一句话概括区别：

- cron 是**时间驱动**——"到点了，执行这个"；
- heartbeat 是**条件驱动**——"醒了，看看有没有事，没事就走"。

常见的错误用法有两种：把所有周期任务都塞进 heartbeat，导致每次心跳都在白烧 token；或者全部用 cron 硬扛，结果"每 10 分钟检查一次有没有新消息"这种任务写出一堆毫无必要的精确调度。

## 做法

### cron：适合有明确触发时间的任务

典型场景：每天 9 点发日报、每小时清理临时文件、固定间隔拉取外部数据。

在 `openclaw.json` 里配 job，或用 `openclaw cron add`（字段名以你装的版本为准）：

```json
{
  "cron": {
    "jobs": [
      {
        "name": "daily-report",
        "schedule": "0 9 * * *",
        "payload": "汇总昨日会话，生成日报发到 Telegram",
        "session": "isolated"
      }
    ]
  }
}
```

要点：独立任务用 isolated session；需要主会话上下文的就投递到 main session。

### heartbeat：适合"条件满足才行动"的任务

典型场景：有未读消息才提醒、磁盘超阈值才告警、不急的例行巡检。

做法：在 workspace 写一个 `HEARTBEAT.md` 清单，间隔在 config 的 heartbeat 配置里调（默认半小时）：

```markdown
# Heartbeat Checklist
- 超过 2 条未回复的用户消息 → 摘要并提醒
- 磁盘使用率 > 85% → 告警
- 都没有 → OK
```

### 决策流程

1. 有明确的触发时间吗？→ cron
2. 是"满足条件才动手"吗？→ heartbeat
3. 又要定时又要判断？→ cron 触发，payload 里让 Agent 先检查条件，不满足直接结束

## 踩坑点

1. **时区**：容器里默认 UTC，`0 9 * * *` 实际是北京时间 17 点。设好 `TZ` 环境变量，或在 job 里显式指定时区。
2. **心跳间隔太短**：每次唤醒都是真实 LLM 调用。5 分钟一次 × 多 agent，费用和 rate limit 都会疼。清单要保证 Agent 能快速判定"没事"。
3. **HEARTBEAT.md 变垃圾场**：清单越长，心跳越贵越慢。原则：单次心跳是"查一眼"，不是"干一小时"，重活一律挪给 cron。
4. **状态别放 session**：isolated session 每次冷启动，没有记忆。任务结果落盘到文件，下次自己读。
5. **重复触发**：同一件事既写了 cron 又写进 HEARTBEAT.md，会收到两条提醒。一个任务只归属一个机制。
6. **补跑语义**：默认情况下错过的 cron 触发不会自动补跑，别假设有 catch-up，先在自己的版本上验证；关键任务要做成幂等的。

## 可复用建议

- 决策口诀：**定时间用 cron，定条件用 heartbeat**。
- `HEARTBEAT.md` 控制在十几行内，每一项都能秒判。
- 有副作用的操作（发消息、删文件）放 cron；heartbeat 只做"发现与上报"。
- 两种机制的运行结果都追加写入日志文件，排障全靠它。
- 上线前用 `openclaw cron run <job>` 手动触发一次验证 payload 行为，别等到第二天早上九点才发现是错的。

## 总结

cron 和 heartbeat 不是二选一的竞品，而是分工：cron 管时钟，heartbeat 管状态。把"什么时候做"交给 cron，把"要不要做"交给 heartbeat，Agent 的自动化才能既准时又省钱。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/c1d9bc920576c731.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/8580d2caff962795.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/0f2c4285eed26d9e.png)

