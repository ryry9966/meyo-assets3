---
title: OpenClaw cron vs heartbeat：定时任务选型与踩坑实录
feedId: 43676
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

在 OpenClaw 里让 Agent 定时干活，主要有两条路：

- **cron**：标准 crontab 表达式，到点触发，执行预定义任务；
- **heartbeat**：Agent 按固定间隔"醒来"，先看一眼状态，由门槛规则判断要不要干活，没事就继续睡。

底层都是定时器，但语义不同：cron 是**时间驱动**，heartbeat 是**时间触发 + 状态驱动**。这个差别决定了各自该用在哪。

## 问题

我早期全用 cron，踩了一串坑：

1. "每 15 分钟检查一次收件箱"的任务，九成轮次的结论是"无事可做"，token 白烧；
2. "每天 9 点发日报"这种对时刻敏感的任务，换成 heartbeat 后触发点会漂移；
3. 长任务没跑完，下一个 tick 又进来，出现并发重复执行。

所以问题不是哪个更强，而是哪类任务配哪种机制。

## 做法

一个够用的决策流程：

1. **触发条件是"某个时刻" → cron**。日报、周报、定点备份，时间本身是需求的一部分。
2. **触发条件是"某件事发生了" → heartbeat**。新邮件才总结、构建失败才告警，时间只是检查频率，状态才决定是否行动。
3. **heartbeat 必须配便宜的前置门槛**。醒来先做轻量检查（文件 mtime、API 状态码、未读数），命中才升级为完整任务，不要每次醒来都做全量推理。
4. **cron 任务写清"无事可做"的出口**，handler 保持幂等，防止重试造成重复副作用。
5. **两种机制都要加锁**，用文件锁或状态标记保证同一任务同一时刻只有一个实例。

配置大致长这样：

```yaml
tasks:
  daily-report:
    type: cron
    expr: "0 9 * * *"
    tz: Asia/Shanghai
  inbox-watch:
    type: heartbeat
    interval: 20m
    gate: inbox_has_unread   # 便宜的前置判断
    lock: true
```

## 踩坑点

- **heartbeat 间隔太短**：5 分钟一次、每次都带完整上下文推理，一天成本远超预期。建议 15–30 分钟起步，观察命中率再调。
- **时区**：容器里默认 UTC，`0 9 * * *` 实际是北京时间 17 点。显式声明 tz，部署后先 `date` 验证一次。
- **错过触发**：休眠或重启后，cron 是否补跑取决于实现；heartbeat 天然"错过就错过"。必须补跑的任务要在任务内部记断点。
- **心跳判断不稳**："要不要干活"不能靠模型自由发挥，要写成明确规则（如"未读 > 1 才行动"），否则会随机空跑或漏跑。
- **任务耗时超过间隔**：cron 会堆积，heartbeat 会重叠，都要设超时上限并配合锁。

## 可复用建议

- 选型口诀：**时刻用 cron，状态用 heartbeat；频率敏感用 cron，成本敏感用带门槛的 heartbeat**。
- 把"是否行动"的规则写成一份 Agent 可读的 checklist 文件，两种机制共用，方便审计和回归。
- 上线前先跑一天 dry-run：只记录"会触发/会行动"，不真执行，用日志校准间隔与门槛。
- 每个定时任务给唯一 ID + 锁文件——这比选哪种机制更重要，是所有定时方案的地基。

## 总结

cron 与 heartbeat 不是替代关系，而是分工：cron 管"确定性时间表"，heartbeat 管"状态感知的按需响应"。实际项目里多数是两个都用，关键动作是把"何时该行动"从时间维度里拆出来，交给门槛规则去判断，而不是让定时器硬扛。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/04adf7df84fd8e72.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/8ce4993979767161.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/0a9b8906f1f41edf.png)

