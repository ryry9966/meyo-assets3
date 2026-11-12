---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 44223
source: 综合讨论
publishedAt: 2026-11-12
---

## 背景

OpenClaw 给 agent 提供了两条"按时间干活"的路径：

- **cron**：传统意义上的定时任务。你指定时间表和一条固定 prompt，到点执行，可选把结果投递到聊天渠道。
- **heartbeat**：心跳。agent 按固定间隔被唤醒一次，自己判断"这轮有没有值得做的事"，没有就静默跳过。

新人的常见误区是两头走极端：要么全塞给 heartbeat（"反正它聪明"），要么把巡检类需求拆成一堆 cron，碎片化到没人维护。

## 问题

我最初用 heartbeat 做每天 8 点的晨会摘要，三天里有两次 8:40 才发——heartbeat 的实际触发时间 = 唤醒时间 + 模型判断，对"准点"没有硬保证。反过来，我又把"帮我盯着测试环境告警"写成了每 10 分钟一次的 cron，跑了一周 token 消耗翻倍，而 90% 的轮次什么都没发生。

## 做法

先用一个判据分流：**"到点必须发生，且产出固定"→ cron；"醒了顺手看一圈，有事才动"→ heartbeat。**

cron 侧示例（字段以你装的版本为准）：

```bash
openclaw cron add \
  --name morning-digest \
  --cron "0 8 * * *" \
  --message "汇总昨天的 git log 和 issue 动态，输出不超过 10 行，发到群里" \
  --channel telegram --to <chat-id> \
  --session isolated
```

heartbeat 侧，在 agent 配置里调唤醒间隔和活跃时段，把"巡检清单"写成短的条件分支，比如：工作区有待办？收到未读通知？测试环境出现新告警文件？让 agent 每轮对照判断，无事则跳过，不要让它自由发挥。

## 踩坑点

1. **cron 的 isolated session 没有主会话上下文**。prompt 必须自包含：路径、数据来源、输出格式写全，否则任务"跑了但产出为空"。
2. **heartbeat 没有准点性**。任何"必须几点发"的需求都该退回 cron，别用提示词去恳求模型守时。
3. **心跳间隔别贪短**。空转同样消耗 token、污染 session 历史，用活跃时段限制夜间唤醒。
4. **cron 失败是静默的**。在 prompt 末尾加一条契约："取不到数据或执行失败时，明确说明原因再投递"，把静默失败变成可观测信号。
5. **时区**。cron 表达式和活跃时段都吃宿主机时区，容器部署记得显式设 TZ。

## 可复用建议

- 口诀：**cron 管"几点必须发生"，heartbeat 管"醒了顺手看一圈"**。
- 新的巡检类需求，先挂在 heartbeat 上观察一两周；发现某项任务其实总在固定时间做，就固化成 cron。
- 给每条 cron prompt 写固定输出契约（格式、长度、失败时的行为），方便下游解析和排障。
- 定期看 cron 任务列表的最近执行状态和 heartbeat 的 token 占用，别让定时任务变成一笔静默成本。

## 总结

cron 和 heartbeat 不是二选一，而是"确定性调度"和"自主巡检"两个互补层。用"准点性 + 产出确定性"这一个判据做分流，大部分选型纠结会自己消失。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/b0d1b4cf0f70c13e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/edda2a6b1f2276b2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/6bd3e1ec521e6208.png)

