---
title: OpenClaw 定时任务怎么选：cron vs heartbeat 的分工实践
feedId: 44061
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景

OpenClaw 里有两套“定时唤醒”机制：**cron** 和 **heartbeat**。cron 是显式排程——写一条表达式，到点触发一个任务；heartbeat 是隐式巡检——agent 按固定间隔醒来，读一遍 HEARTBEAT.md，自己判断有没有值得做的事。两者都能“定时跑 agent”，但语义完全不同，选错会直接体现在 token 账单和消息体验上。

## 问题

最常见的错是混用：把“每小时看一眼收件箱，有事才告诉我”写成 cron，结果每小时空跑一次全量对话，什么都没发生也烧一次 token；反过来，把“每天 9 点必须发日报”交给 heartbeat，靠 agent 自觉守时，偶尔就静悄悄错过。核心矛盾在于：**准点必达**和**有事才说**是两种需求，不能用同一种机制硬套。

## 做法

1. **先分类。** 列出所有定时需求，逐条标注三个属性：是否必须准点、是否无条件执行、失败后果是什么。
2. **必达类进 cron。** 日报、定时提醒、备份触发这类“到点必须发生”的任务：

```bash
openclaw cron add --name daily-digest \
  --cron "0 9 * * *" \
  --message "汇总昨天的会话要点，发到 #daily 频道"
```

写完用 `openclaw cron run <id>` 手动触发一次验证产出，再确认时区语义（表达式按服务器时区解释）。

3. **巡检类进 heartbeat。** 配置里设 `heartbeat.intervalMinutes`（比如 30），判断规则写进工作区的 HEARTBEAT.md。关键是写“触发条件”而不是开放指令，例如“仅当出现未处理告警或用户留言时输出，否则保持静默”。token 敏感的话，给 heartbeat 配轻量模型做预判，只有判出需要处理才唤醒主模型，空拍成本能降一个量级。
4. **分开会话和通道。** cron 结果投递到专用频道；heartbeat 默认落在主会话，别让巡检输出刷掉日常对话。
5. **观察一周再调参。** 看 heartbeat 的实际触发频率和“空拍”比例，回调间隔和 HEARTBEAT.md 的严格程度。

## 踩坑点

- **空跑烧 token：** 把条件判断写进 cron 的 message 并不省钱，agent 仍然全量跑。巡检类务必走 heartbeat。
- **心跳太吵：** HEARTBEAT.md 写得太泛，agent 每一拍都“觉得有事”。要加硬约束，比如“每天最多提醒 2 次”。
- **时区坑：** 服务器是 UTC 时，9 点实际在 17 点触发。部署后先手动跑一次对时。
- **错过不补：** 离线期间错过的 cron 任务不一定补跑，重要任务要么确认补跑语义，要么另加自检。
- **心跳静默 ≠ 正常：** heartbeat 自己挂了不会报错。可以用一条每日 cron 反问 agent“上一拍心跳是什么时候”——用 cron 监控 heartbeat，形成互补。

## 可复用建议

- 一条经验法则：**准点必达用 cron，有事才说用心跳**；同一个需求不要两边都挂。
- cron 任务四要素模板：名称、时间、目标通道、失败处理。
- HEARTBEAT.md 三原则：条件明确、禁止开放式指令、带频率上限。
- 每月 review 一次 `openclaw cron list`，清掉僵尸任务。

## 总结

cron 和 heartbeat 不是替代关系，而是分工：cron 负责**确定性时间表**，heartbeat 负责**低成本的存在感检查**。按“准点、必达、无条件”三个维度分类，配合分开通道、手动验证、定期 review，两套机制就能稳定共存，而不是互相打架。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/65c8a259543cd958.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/0c659e31d1e23bc1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/f3ff0e6aa95fb292.png)

