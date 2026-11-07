---
title: OpenClaw 定时任务怎么选：cron 管准点，heartbeat 管盯梢
feedId: 43700
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

OpenClaw 里有两套让 agent“按时干活”的机制：**cron** 和 **heartbeat**。刚上手时很容易混淆——两者都能定时触发，但设计意图完全不同。我把自己的选型过程整理成这篇笔记，供社区参考。

## 两种机制的本质区别

**cron 是“到点就执行”**：你给一个 cron 表达式和一段固定 prompt，到点后由调度器发起一次独立的 agent 会话。执行内容基本确定，比如“每天早上 9 点汇总仓库新增 issue，推送到 Telegram”。

**heartbeat 是“到点看一眼”**：agent 按固定间隔（默认约 30 分钟）被唤醒，读取 workspace 里的 `HEARTBEAT.md` 检查清单，由模型自行判断这一轮有没有事要做。没事就静默跳过，有事才产出消息或调用工具。

一句话概括：**cron 的决策权在你（写在 prompt 里），heartbeat 的决策权在 agent（读完清单自己判断）**。

## 我的选择标准

按三个维度判断：

1. **触发精度**：必须准点的事件用 cron——日报、周报、定时提醒、固定时间的摘要。误差半小时不可接受的任务，只能 cron。
2. **是否有明确检查清单**：heartbeat 适合“盯状态”类任务——检查某个目录的新文件、确认服务存活、轮询长任务是否完成。特点是“大多数时候没事做”。
3. **成本敏感度**：heartbeat 每次唤醒都是一次真实的模型调用，哪怕结论是“无事发生”，也在持续烧 token；cron 只在你定义的时间点花钱。

## 实际配置步骤

**cron**：直接在会话里用自然语言描述，让 agent 生成任务：

```text
建一个 cron 任务：每天 09:00 汇总目标仓库新增 issue，推送到 Telegram
```

生成后用 `openclaw cron list` 核对表达式和 payload，必要时 `openclaw cron edit` 修正。

**heartbeat**：编辑 workspace 下的 `HEARTBEAT.md`，把要盯的事项写成条件清单：

```markdown
- 检查 ~/data/inbox 是否有新文件，有则整理归档
- 若 deploy.lock 存在超过 1 小时，提醒我
- 其他情况不输出任何内容
```

再把配置里的心跳间隔调到 30–60 分钟，重启生效。

## 踩坑点

1. **HEARTBEAT.md 写成“每次都要汇报”**：我早期在清单里写了“汇报当前状态”，结果每半小时收到一条废话消息。每条都应该是条件句：“仅当 X 发生时才动作”。
2. **heartbeat 唤醒可能被跳过**：设备休眠或 agent 离线期间的唤醒不一定补跑。强依赖准点的任务绝不要放 heartbeat。
3. **cron 的时区**：调度默认看宿主机时区，VPS 在 UTC 时“早上 9 点”实际是下午 5 点，部署前先确认。
4. **心跳间隔压太短**：低于 15 分钟，token 消耗翻倍且大多数唤醒一无所获，除非任务真的高频。

## 可复用建议

- 定时产出类（报告、摘要、提醒）→ cron；状态巡检类（监控、看门狗、轮询）→ heartbeat。
- 心跳间隔宁长勿短：先从 60 分钟跑一周，观察消息频率再收紧。
- `HEARTBEAT.md` 每条都写清“触发条件 + 动作 + 静默条件”三要素。
- 可靠性要求高的 cron 任务，让 agent 失败时也主动推送通知，而不是静默。

## 总结

cron 和 heartbeat 不是竞争关系，而是分工：**cron 负责“你知道什么时候该做的事”，heartbeat 负责“你不知道什么时候会发生、但需要有人盯着的事”**。把决策权限划分清楚，agent 的定时行为才会既准时又安静。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/3be1482939f21c65.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/8b12d7fd5d80cc9e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/a6d8f4af3a471d14.png)

