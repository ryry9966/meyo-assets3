---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 43706
source: 综合讨论
publishedAt: 2026-11-07
---

# OpenClaw 的 cron vs heartbeat：两种定时任务怎么选

## 背景

OpenClaw 的 agent 默认是被动应答：你不发消息，它不动。但真实使用中大量需求是主动的——每天早上推一份晨报、盯一下收件箱、定期清理临时目录。OpenClaw 内置了两套触发机制：**cron**（定时任务）和 **heartbeat**（心跳）。两者都能让 agent"自己动起来"，但设计意图完全不同。

## 问题

社区里常见的误用有两种：把所有定时需求都塞进 heartbeat，导致 token 消耗暴涨；或者用 cron 去做"盯着某个状态"的事，结果要么频繁空跑，要么错过时机。选错机制，多花钱是小事，任务语义直接歪掉才是大事。

## 两种机制的本质区别

- **cron 是时间驱动**：到点必触发，触发内容是你写死的 prompt。适合"某时某刻做某事"。
- **heartbeat 是状态驱动**：按固定间隔（默认 30 分钟）把一条心跳消息送进主会话，agent 醒来看一眼 `HEARTBEAT.md` 的检查清单和当前上下文，自己决定要不要行动。不行动就回一个极简占位，成本低但不是零。

一句话：**cron 是闹钟，heartbeat 是巡逻**。

## 做法

**用 cron 做确定性任务。** 在 `openclaw.json` 的 `cron.jobs` 里定义，或用 CLI 添加：

```json
{
  "name": "morning-digest",
  "schedule": { "kind": "cron", "expr": "0 9 * * *", "tz": "Asia/Shanghai" },
  "payload": { "kind": "agentTurn", "message": "汇总昨天的 GitHub 通知和 RSS，输出晨报" },
  "sessionTarget": "isolated",
  "delivery": { "channel": "telegram", "to": "me" }
}
```

要点：显式写 `tz`；长任务用 `isolated` 会话，不占主对话；需要推送就配 `delivery`。schedule 除了 cron 表达式，还支持 `every` 固定间隔和 `at` 一次性时间点。

**用 heartbeat 做状态巡检。** 配置间隔，然后在 workspace 根目录放 `HEARTBEAT.md`：

```json
{ "agents": { "defaults": { "heartbeat": { "every": "30m" } } } }
```

清单条目要写成"可观察、可判断"的事，比如"若 `~/inbox` 有未处理文件，整理归档"。控制在 3~5 条以内。

## 踩坑点

1. **heartbeat 不是免费的。** 每次 tick 都是一次模型调用。把间隔压到 5 分钟，成本约为默认值的 6 倍，而绝大多数 tick 的结论是"没事"。
2. **HEARTBEAT.md 写成了愿望清单。** 条目含糊（"帮我关注一下项目动态"），agent 每次 tick 都会过度反应。条目必须有明确的触发条件。
3. **cron 时区默认跟 gateway 走。** 容器里往往是 UTC，`0 9 * * *` 实际是北京时间下午五点。务必显式指定。
4. **cron 错过不补跑。** 机器关机或 gateway 重启窗口正好命中触发点，这次就没了。对"必须发生"的通知，考虑加自检任务兜底。
5. **isolated 会话没有主对话记忆。** 别指望 cron 任务记得你昨天聊过的偏好，要么写进 prompt，要么让它去读固定文件。
6. **heartbeat 的 tick 在主会话忙时会被跳过。** 这是特性不是 bug，别拿它做秒级精度的调度。

## 可复用建议

- 判断标准一条：**用户预期"到点就有结果"→ cron；预期"有事你会告诉我"→ heartbeat。**
- 两者可以组合：heartbeat 发现重复出现的异常时，让 agent 建议你"要不要加一条 cron 把它固化"，这是很自然的演进路径。
- 心跳 tick 尽量轻：清单短、间隔不激进、必要时给 heartbeat 单独配更便宜的模型；再用 `activeHours` 避开深夜空转。
- 定期用 `openclaw cron list` 和运行日志审计实际触发情况，比事后猜测靠谱。

## 总结

cron 和 heartbeat 不是二选一，而是分工。cron 管"确定性"，heartbeat 管"可能性"。把必须发生的事交给 cron，把"不确定要不要发生"的事交给 heartbeat，token 账单和任务可靠性都会好很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/50eca23f4ddcf882.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/eb389723fad20e0c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/3385d09fbd259a66.png)

