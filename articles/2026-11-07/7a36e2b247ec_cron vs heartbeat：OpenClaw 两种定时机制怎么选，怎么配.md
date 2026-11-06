---
title: cron vs heartbeat：OpenClaw 两种定时机制怎么选，怎么配
feedId: 43666
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

在 OpenClaw 里，一切“自动化”最终都是触发一次 agent 运行，但触发方式主要有两种：

- **cron**：经典 cron 表达式驱动的调度器，跑在 gateway 进程里，到点执行，默认在**隔离会话**（isolated session）中运行，支持把结果投递到指定渠道。
- **heartbeat**：按固定间隔（默认 30 分钟）向主会话注入一个轻量的心跳 prompt，由 agent 自己判断“这一轮要不要干活”。

## 问题

真实需求往往横跨两类：

- “每天早上 8 点发早报”——时间和内容都确定，cron 天然合适；
- “帮我盯着收件箱和 CI，有值得处理的事再叫我”——触发条件无法用时间表达，需要 agent 运行时看状态，heartbeat 合适。

选错的代价很具体：用 heartbeat 做定点早报，实际触发会在窗口期内漂移，而且每个间隔都为它付一次 token；反过来用 cron 逼近“有新邮件就跑”，你得堆一堆表达式，依然覆盖不全。

## 做法

**Step 1：先分类。** 判断“何时执行”由谁决定——写在配置里的是 cron，agent 运行时判断的是 heartbeat。

**Step 2：cron 骨架**（flags 以你当前版本为准）：

```bash
openclaw cron add \
  --name "morning-digest" \
  --cron "0 8 * * *" \
  --session isolated \
  --prompt "读取今日日历与未读消息，输出 10 行以内早报" \
  --deliver --channel telegram
```

要点：prompt 写成自包含（隔离会话没有主对话上下文），明确投递渠道，重活设 timeout。

**Step 3：heartbeat 骨架**：

```json5
{ heartbeat: { every: "30m" } }
```

然后在 AGENTS.md 里写清“什么情况才值得打扰我”：只在 CI 失败、有人紧急联系、日历冲突时行动，其余一律安静返回 ack。

**Step 4：验证。** cron 用 `cron list` 核对下次触发时间，手动 run 一次走通投递链路；heartbeat 先把间隔临时调到 5 分钟观察几轮，再调回正常值。

## 踩坑点

1. **heartbeat 全局只有一个间隔**，做不到“任务 A 每 5 分钟、任务 B 每 2 小时”——那是 cron 的活。
2. **隔离会话失忆**：别在 isolated cron prompt 里写“我们刚才聊的内容”；确实需要上下文就显式塞进 prompt，或改用 main session（慎用，会污染主对话）。
3. **时区**：容器里 gateway 常默认 UTC，cron 表达式和 activeHours 会整体错位，先固定 TZ。
4. **heartbeat 的两极**：agent 学会永远 ack → 零产出；永远觉得有事 → 变骚扰加 token 黑洞。判断标准要写成可执行的白名单，别写“看着办”。
5. **双投递**：cron 开了 `--deliver`，agent 又自己调发消息工具，会出现重复消息，二选一。

## 可复用建议

- 一句话选型：**时间驱动用 cron，状态驱动用 heartbeat；精确到分钟用 cron，“帮我盯着”用 heartbeat。**
- heartbeat 从 30~60 分钟起步，观察 token 消耗再收紧。
- cron 的 prompt 按“给新同事派活”的标准写，自包含、有边界、有产出格式。
- 每月清理一次 `cron list`，失效任务及时删。

## 总结

cron 和 heartbeat 不是竞争关系，是分工关系：cron 管“日历上确定的事”，heartbeat 管“环境里不确定的事”。主干任务交给 cron、监控判断交给 heartbeat，再配一份严格的打扰白名单，整套自动化就会既准时、又安静。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/e687a711f475c910.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/8550779c2ff15607.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/d243afa9bb12c752.png)

