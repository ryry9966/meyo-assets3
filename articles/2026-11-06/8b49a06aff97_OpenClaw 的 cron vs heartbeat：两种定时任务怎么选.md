---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 43570
source: 综合讨论
publishedAt: 2026-11-06
---

## 背景

OpenClaw 里有两种原生的"让 Agent 定时干活"的机制：**heartbeat**（周期性唤醒，读工作区的 HEARTBEAT.md 清单）和 **cron**（crontab 风格的定时任务）。表面上都是"隔段时间触发一次模型"，但设计意图和成本模型完全不同。我见过不少用反的案例，结果要么是 token 白烧，要么是关键任务错过时间点。

## 问题

真实需求通常分两类：

- "每天早上 9 点，把昨天部署摘要推给我" —— 时间点固定，结果必须送达；
- "盯一下磁盘、CI 和收件箱有没有异常，有事再叫我" —— 时间不固定，按需触发，无事保持安静。

两种机制各管一头，混着用就会出问题。

## 机制差异

| 维度 | heartbeat | cron |
|---|---|---|
| 触发方式 | 固定间隔（默认 30m） | cron 表达式 / 间隔 / 一次性 at |
| 谁决定做什么 | Agent 每次醒来按清单自行判断 | 计划和提示词在配置里写死 |
| 会话 | 主会话，带对话上下文 | 可选 isolated，独立上下文 |
| 输出 | 默认静默（HEARTBEAT_OK 即不出声） | 可选 `--announce` 推送到频道 |
| 持久化 | 配置 + 清单文件 | 任务列表持久化，重启不丢 |

一句话规则：**有明确钟点、动作固定的走 cron；没有固定时间、需要判断、默认沉默的走 heartbeat。**

## 做法

heartbeat 三步：

1. 配置里设 `agents.defaults.heartbeat.every: "30m"`；
2. HEARTBEAT.md 只写短条件，例如"磁盘占用 > 85% 时汇报一次，否则保持安静"；
3. 确保默认分支是沉默，避免无事唤醒、有事刷屏。

cron 三步：

1. `openclaw cron add --name daily-digest --cron "0 9 * * *" --session isolated --announce --message "总结昨日部署与失败项"`（参数名随版本略有出入，以 `openclaw cron --help` 为准）；
2. 先用 `openclaw cron run <id>` 手动干跑一次，确认输出格式和投递路径；
3. `openclaw cron list` 核对计划与下次触发时间。

## 踩坑点

1. **heartbeat 间隔压到 1–5 分钟**：每次唤醒都是真实模型调用，回复 HEARTBEAT_OK 也计费。30 分钟是合理起点，不需要模型判断的事不该走 heartbeat。
2. **清单写成待办列表**：Agent 每次醒来都执行一遍，疯狂刷消息。清单应该是"条件 → 动作"，不是"要做的事"。
3. **cron 死守主会话**：任务提示词混进主对话上下文，污染记忆，也被主对话干扰。一次性任务一律 isolated。
4. **cron 忘了 announce**：任务跑了，结果没有任何去处，你以为它没执行。需要人看的结果务必配投递。
5. **时区与错过触发**：表达式按网关主机时区解析；网关重启后的错过任务，用 miss 策略决定跳过还是补跑。
6. **把 heartbeat 当实时**：它是轮询不是事件。"一发生就触发"请走频道 webhook / 事件类插件，缩短间隔救不了延迟。

## 可复用建议

- 默认配置：heartbeat 30m + 短条件清单；凡有明确钟点的任务进 cron；
- 一条 cron 任务只做一件事，isolated + announce 是标准搭配；
- 新 cron 任务先手动干跑再放行；
- 每周看一次 token 用量，heartbeat 占大头就拉长间隔，或把不需要判断的任务挪走；
- HEARTBEAT.md 是决策文件，不是待办堆放处，越短越好。

## 总结

两者的本质区别是"谁决定何时行动"：cron 是你决定（时刻表写在配置里），heartbeat 是 Agent 决定（醒来后按清单判断）。稳定组合是 cron 管钟点驱动，heartbeat 管状态驱动，别让一方替另一方干活。当你发现 heartbeat 清单里堆了一堆固定时刻的任务，就该搬家了；反过来，如果 cron 任务全是"看看有没有异常"，那 heartbeat 更合适。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/a1cb645f756c86c0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/835fd1db395cebb3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/0516c500c152fc3e.png)

