---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 44161
source: 综合讨论
publishedAt: 2026-11-12
---

## 背景

用 OpenClaw 做自动化一段时间后，几乎都会遇到同一类需求：「每天早上 9 点给我一份日报」「每隔半小时看看某个目录或收件箱有没有新东西」。框架里能实现这两件事的入口有两个：**cron job** 和 **heartbeat**。表面上都是「定时唤醒 agent」，但设计意图完全不同。

## 问题

混用两类需求会出现典型事故：

- 用 heartbeat 做「必须准点」的任务——agent 正在忙主会话，提醒晚了四十分钟；
- 用 cron 做「看状态再决定」的任务——只能写死频率，要么频繁空跑烧 token，要么间隔太长错过窗口。

先把两者分清楚：

- **cron**：配置在 `openclaw.json` 的 `cron.jobs`（或让 agent 用 cron 工具自己添加）。支持 cron 表达式、固定间隔、单次 `at` 三种 schedule，到点把 payload 作为 prompt 投递给 agent，投递方式可选 `announce`（推送到指定频道）或 `isolated`（独立会话静默执行）。它是纯调度器，**时间是唯一触发条件**。
- **heartbeat**：主 agent 周期性被唤醒（`agents.defaults.heartbeat.every`，默认 30 分钟），每次醒来读工作区的 `HEARTBEAT.md`，自行判断有没有事要做，没有就跳过。触发条件是时间，但**决策权在 agent**。

## 做法

1. **时间驱动 → cron。** 写清 `name`、`schedule`、`payload`、`delivery`。例如每日 9 点日报，`schedule` 用 `daily` 并显式指定 `tz: Asia/Shanghai`，`delivery.mode` 设 `announce`。临时任务直接在对话里让 agent 调 cron 工具加一条 `at` 单次任务，比手写配置快。
2. **状态驱动 → heartbeat。** 在 `HEARTBEAT.md` 里用短句列「待检查条件」，每条一行、可判断、可执行，比如「若 `~/reports` 存在超过 24h 未同步的文件，则提醒我」。间隔建议不小于 30 分钟。
3. **重活别放 heartbeat。** heartbeat 跑在主会话里，长回复会污染主上下文；需要多轮执行的任务改用 cron + `isolated` 会话。

## 踩坑点

- **heartbeat 不是免费的**：每次心跳都是一次真实模型调用，即使结果是 skip。`HEARTBEAT.md` 写成大段散文会让每次唤醒都变贵，务必条目化、短句。
- **cron 没有补跑**：网关在触发点不在线，任务直接跳过，重启后不回放。重要的每日任务确认 daemon 常驻。
- **时区**：cron 默认用系统本地时区，跨时区部署务必显式写 `tz`。
- **投递模式选反**：`announce` 会推到你的聊天窗口（适合日报），`isolated` 静默执行（适合后台任务），选反了要么刷屏，要么「任务到底跑没跑」无从确认。
- **心跳间隔小于 agent 常规响应耗时会叠跑**，主会话出现排队。

## 可复用建议

- 一句话决策：**到点必做的事交给 cron，「有事才做」的事交给 heartbeat。**
- 把 `HEARTBEAT.md` 当 checklist 维护，不超过一屏；跑一段时间后，把从未触发过的条目删掉或降频。
- 用 cron 做兜底：即使主逻辑在 heartbeat 里，也可以加一条低频 cron 做每日汇总，保证有确定性输出。
- 排查顺序：先 `openclaw cron list` 确认任务存在，再核对 delivery 通道，最后才查 heartbeat 配置。

## 总结

cron 和 heartbeat 不是竞争关系，而是互补：cron 是你给 agent 的日程表，heartbeat 是 agent 给自己设的巡检岗。把「时间触发」和「状态触发」分开建模，token 消耗和任务可靠性都会明显改善。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/b5862c10906ed060.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/cfde159bbb2efda0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/0395d75af93f78af.png)

