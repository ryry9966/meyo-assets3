---
title: cron 还是 heartbeat？OpenClaw 定时任务选型实录
feedId: 43925
source: 综合讨论
publishedAt: 2026-11-09
---

# 背景

OpenClaw 里有两套"定时"机制，新人很容易混着用：

- **cron**：经典 cron 表达式驱动，到点触发一次 agent 会话，执行你预设的 payload。适合日报、定时提醒、周期性抓取这类"时间到了就干"的任务。
- **heartbeat**：按固定间隔（默认 30 分钟）给 agent 发一个心跳 prompt，agent 读取工作区的 `HEARTBEAT.md`，自行判断"有没有事要做"，没事就回 `HEARTBEAT_OK` 保持沉默。

一个是闹钟，一个是巡逻。

# 问题

实际用下来最常见的两类翻车：

1. 把 heartbeat 当精确调度器："每天 8 点发日报"变成 8:03、8:47 随机漂移，而且清单写太长，token 消耗翻了几倍。
2. 把所有逻辑塞进 cron payload，但 cron 是隔离会话，没有主对话上下文，任务拿到手一脸茫然。

# 做法

选型一句话：**"几点做"用 cron，"该不该做"用 heartbeat。**

**用 cron：**

1. 列出所有时间驱动的任务：日报、周报、定时开关服务、每日抓取。
2. 逐个注册 job，payload 必须自包含（字段名以你部署版本的 schema 为准）：

```json
{
  "name": "morning-digest",
  "schedule": "0 8 * * *",
  "prompt": "读取 ~/notes/inbox.md，汇总为不超过 5 条的要点，发到 Telegram"
}
```

3. 想清楚错过怎么办：日报类错过就跳过，监控类才考虑补跑。

**用 heartbeat：**

1. `HEARTBEAT.md` 只写"条件 + 动作"，比如"如果日历显示 15 分钟内有会议，提醒我"。别写成任务列表。
2. 按需调间隔，从默认 30 分钟改成 `15m` 或 `1h`。
3. 保持"没事回 HEARTBEAT_OK"的习惯，静默本身就是特性。

**混合模式（推荐）：** heartbeat 当哨兵，发现条件满足后由 agent 通过 cron 工具登记一次性任务去干重活。心跳只负责扣扳机，不负责搬砖。

# 踩坑点

- **heartbeat 成本是隐性的**：每个 tick 都是一次模型调用，哪怕最后只回了个 OK。清单压到 10 行以内，观察一两周用量再调间隔。
- **心跳会漂移**：机器休眠、容器重启后，间隔从唤醒时刻重新计算，别指望它准点。
- **cron 时区坑**：表达式按 gateway 主机时区解释，Docker 里显式设 `TZ`，否则"8 点"可能是 UTC 的 8 点。
- **cron 会话无记忆**：payload 里别写"继续刚才的讨论"，输入文件路径、输出渠道全部写死。
- **心跳重入**：上一 tick 的长任务没跑完，下一 tick 又进来。给这类检查加状态标记，或直接调大间隔。

# 可复用建议

- cron job 命名带用途和频率（如 `daily-08-digest`），半年后你还能看懂。
- 每季度清一次 job 列表，过期任务及时删。
- heartbeat 清单里每一项都问自己：这个判断值得一次模型调用吗？不值的挪进 cron。
- 先用 heartbeat 低成本试跑，观察稳定后再固化成 cron，是安全的迁移路径。

# 总结

cron 是闹钟，heartbeat 是巡逻。前者给你确定性，后者给你环境感知。大多数稳定运行的实例，最终形态都是少数几条 cron 承担确定性任务，一条精简的 heartbeat 兜底巡检——别让任何一个机制去干另一个的活。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/e76d14eebc4f57b3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/569aadbbae9c462b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/f71fd8295bd9cd8e.png)

