---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 43683
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

用 OpenClaw 一段时间后，你大概率会遇到同一个需求："能不能让 agent 自己动起来，别每次都等我开口。" 官方给的路有两条：**cron jobs** 和 **heartbeat**。两者都由时间驱动，但设计意图完全不同——一个是日程表，一个是值班员。混着用，结果通常是消息轰炸或任务漏跑。

## 问题

典型的两难：

- 每天早上 9 点发一份日报，该用哪个？
- 想让 agent"盯着"某个后台任务，跑完了提醒我，该用哪个？

把 heartbeat 当闹钟用，你会发现它不保证准点；把 cron 当轮询器用，每 5 分钟建一个任务既啰嗦，也表达不了"跑完才提醒"这种条件。根源在于两者的触发语义不同。

## 做法

### cron：适合"确定的时刻"

cron 的触发条件是一个时间表达式，到点必跑，有运行记录，payload 是你预先写好的消息。示意（参数以你版本的 `openclaw cron --help` 为准）：

```bash
openclaw cron add --name daily-report \
  --cron "0 9 * * *" \
  --message "汇总昨天的会话要点，发到群里" \
  --deliver --channel telegram
```

要点：

- 一次性任务用 `--at` 指定绝对时间，跑完自动清理。
- `session` 选 `isolated`，别挂在主会话上，避免输出和正在聊的内容搅在一起。
- 改任务优先走 `openclaw cron edit / list / run`，少手改 `jobs.json`。

### heartbeat：适合"不确定的条件"

heartbeat 是周期性"醒来"：按 `HEARTBEAT_INTERVAL`（默认 30 分钟，也可在 `openclaw.json` 里调）触发一次轻量推理，agent 读一遍 workspace 里的 `HEARTBEAT.md`，判断有没有事要做。没事就回 skip 标记（默认 `HEARTBEAT_OK`），不投递；有事才开口。

实践上，把"值班时要看什么、什么条件下做什么"写成两三行清单放进 `HEARTBEAT.md`，例如"检查后台下载任务是否完成，完成后通知我"。重活交给脚本或工具，agent 只看结果再决策。

### 一条判断线

- 触发条件能写成时间表达式 → cron
- 触发条件是"某个状态/文件/结果变了" → heartbeat
- 要投递到渠道、要审计每次执行 → cron
- 高频、轻量、条件式检查 → heartbeat

## 踩坑点

1. **heartbeat 当闹钟**：gateway 重启、间隔漂移，9 点的任务可能 9 点 17 分才跑。精确时刻一律 cron。
2. **HEARTBEAT.md 越写越长**：每次心跳都要读和判断，清单越长 token 越贵。判断逻辑压到两三行，检查逻辑下沉到脚本。
3. **skip 标记失效**：心跳回复了正常文本就会被投递出去，变成定时骚扰。验证模型严格输出 `HEARTBEAT_OK`，必要时在 HEARTBEAT.md 里重申规则。
4. **cron 时区**：容器里默认 UTC，按本地时间写的表达式会整体错位，配置里显式指定 timezone。
5. **cron 挂主会话**：日报内容和对话流混在一起。隔离 session + 单独指定投递渠道。
6. **CLI 和手改 jobs.json 混用**：运行中的 gateway 可能覆盖手写内容，统一走 CLI。

## 可复用建议

- 一句话原则：**确定的时刻用 cron，不确定的条件用 heartbeat。**
- 组合用法：cron 负责"产出"（抓数据、跑批、发日报），heartbeat 负责"巡检"（检查 cron 的产物、盯后台任务、过待办清单），心跳只做"要不要叫人"的判断。
- 成本控制：心跳间隔别低于 15 分钟，除非确实需要；每类检查做成幂等脚本，agent 只读结论。
- 上线前先 `openclaw cron run` 手动触发一次，确认 session 行为和投递渠道符合预期，别等第二天早上才发现发错了地方。

## 总结

cron 是日程表，heartbeat 是值班员。别让日程表看状态，也别让值班员看表。把触发语义分开、把重活下沉到脚本、把投递目标定清楚，agent 的主动性才可控，账单也可预期。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/1e5549ef325710e2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/ad431c886526c832.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/270246647aefe8b6.png)

