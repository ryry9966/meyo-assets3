---
title: cron vs heartbeat：OpenClaw 定时任务的两种选法
feedId: 44321
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

OpenClaw 里有两套看起来很像、实际上分工完全不同的“定时”机制：

- **cron**：标准定时调度。到点执行一条预置 prompt，时间用 cron 表达式精确控制。
- **heartbeat**：agent 周期性自唤醒（默认 30 分钟一次），读取工作区的 `HEARTBEAT.md`，自己判断这轮有没有事可做。

## 问题

实际使用中最常见的困惑是：两边都能做“定时推送”、“周期检查”，到底写在 cron 里还是 `HEARTBEAT.md` 里？写错了会出两种典型事故：

1. 把“每 15 分钟检查一次通知”写进 cron——每次都是一次完整会话调用，token 消耗线性暴涨。
2. 把“每天 9:00 发日报”写进 `HEARTBEAT.md`——heartbeat 是“自启动起每 N 分钟”的间隔逻辑，不对齐整点，重启后计时归零，实际可能在 9:07 触发，日报时间天天漂。

## 做法：按“时间驱动还是条件驱动”选

判断标准一句话：**有精确时刻 → cron；有状态要轮询 → heartbeat。**

cron 适合：

- 日报、周报等对齐整点的输出；
- 定时提醒（“每天 21:30 提醒我收拾工位”）；
- 定期抓数据落盘（抓天气、汇率写进文件，供主会话随时读）。

heartbeat 适合：

- 收件箱、PR、告警类“有事才说”的巡检；
- 状态类维护：磁盘水位、长任务是否跑完；
- 需要上下文判断的场景——heartbeat 跑在主会话里，能结合最近的对话做判断，这是 cron 隔离会话给不了的。

配置要点：

1. **cron**：通过 cron 工具新增任务（存储在 jobs.json），写清 schedule + prompt。默认跑在隔离会话，不污染主上下文；需要发消息进主会话时，显式指定 target。
2. **heartbeat**：编辑 `HEARTBEAT.md`，把检查项写成清单式指令，并且必须写明“无事可做时只回复 HEARTBEAT_OK”。

## 踩坑点

1. heartbeat 默认每 30 分钟醒一次，**即使 `HEARTBEAT.md` 是空的也会调用模型**。不用巡检就干脆关掉 heartbeat，别留空文件。
2. 不写 HEARTBEAT_OK 规则的话，agent 每次醒来都可能往聊天里发消息，一天几十条，很快被自己人静音。
3. cron 的隔离会话没有主会话记忆，prompt 里要自带全部信息（路径、目标频道、格式要求），别指望它“记得”昨天的对话。
4. 高频 heartbeat（如 5 分钟）+ 长检查清单，成本是乘法关系。检查项要写“先做廉价判断，再做昂贵动作”。
5. heartbeat 的间隔从启动时刻起算，重启即归零，任何要整点对齐的事都不能依赖它。

## 可复用建议

- 成熟的分工模式：**cron 管钟点，heartbeat 管巡逻**。例如 cron 每 6 小时抓一次数据落盘，heartbeat 巡检时发现数据异常再主动开口。
- `HEARTBEAT.md` 里每条检查项写三要素：触发条件、执行动作、频率上限（如“每小时最多提醒一次”），避免重复轰炸。
- 临时加巡检项只动 `HEARTBEAT.md`，改时间只动 cron，两套配置互不牵连，回滚也容易。
- 观察一周 token 用量再调 heartbeat 间隔，多数场景最后落在 30–60 分钟。

## 总结

cron 和 heartbeat 不是竞争关系，是两种触发哲学：cron 是“时间说了算”，heartbeat 是“状态说了算”。精确时刻交给 cron，条件巡检交给 heartbeat，写清 HEARTBEAT_OK 规则和隔离会话的边界，定时自动化基本就不会翻车。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/cb3dee738262e24b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/895e433ffcfdd158.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/a6feffec09b4b7f5.png)

