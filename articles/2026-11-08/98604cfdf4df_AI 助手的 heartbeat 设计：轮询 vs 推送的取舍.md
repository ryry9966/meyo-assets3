---
title: AI 助手的 heartbeat 设计：轮询 vs 推送的取舍
feedId: 43831
source: 综合讨论
publishedAt: 2026-11-08
---

## 背景

OpenClaw 的 agent 是长期驻留的：gateway 挂着各种 channel，模型只在被唤醒时才干活。所谓 heartbeat，就是一个定时器——每隔 N 分钟往 agent session 注入一条轻量提示，让 agent 自查一次"有没有值得做的事"，没有就回 `HEARTBEAT_OK` 静默掉。它的本质是轮询。另一条路是推送：webhook、channel 事件、MCP 通知，状态源变了主动来敲门。两者怎么配比，是做自动化时绕不开的取舍。

## 问题

实际用下来最常见的三个坑，本质都和"该轮询还是该推送"有关：

1. **把 heartbeat 当 cron 用**。塞重活进去，session 上下文越滚越大，token 烧得快，还污染主对话。
2. **空 tick 占比过高**。状态源一天没几次变化，却每 5 分钟白跑一次模型调用，纯烧钱。
3. **推送不可靠**。webhook 在 gateway 重启窗口丢了，事件永远没被处理，而轮询兜底又没配，任务悬空。

## 做法

我的原则是：**推送保时效，轮询做对账，幂等防重复。** 具体步骤：

1. **分层判定**。凡是状态源自己能发 webhook / 事件（git push、CI 完成、文件 watcher），一律走推送。推送只负责"敲一下"：把事件写进 workspace 的待办文件，再唤醒 agent。
2. **heartbeat 降级为兜底**。interval 保守设置（默认 30 分钟够用，急事靠推送，别图省事拉到 1 分钟）。heartbeat prompt 保持极小，明确指示：无事可做就回 `HEARTBEAT_OK`，不执行重任务。
3. **幂等键落盘**。每个事件带唯一 key，workspace 里存 last-processed 标记（或 ETag / mtime）。推送和 heartbeat 命中同一事件时先比对标记，处理过就跳过——这是防双重消息的关键。
4. **对账**。heartbeat 的检查逻辑就是"拉一次状态源，和落盘标记比对，补处理漏掉的事件"。这样推送丢了也有下限。
5. **可观测**。每个 tick 记日志：耗时、是否产生动作。统计空 tick 比例，持续偏高就该提高 interval 或改成推送。

## 踩坑点

- **上下文污染**：在 heartbeat 里跑长任务，几轮就把 session 撑爆。重活拆到独立 cron session，heartbeat 只做"发现 + 转交"。
- **时间不精确**：容器节流、笔记本休眠都会让 interval 漂移，别在 heartbeat 上建任何 SLA。
- **重启丢事件**：推送没有重放就等于会丢。要么事件桥带队列，要么接受轮询兜底的成本。
- **静默时段**：没配 activeHours，agent 半夜推"今日待办"，体验很差。心跳类消息要设投放时段和投放目标。

## 可复用建议

- 一句话选型：状态源可控 → 推送；不可控或事件易丢 → 轮询兜底；生产环境 → 两者叠加。
- heartbeat 是门卫不是执行者：prompt 越小越好，职责只有"判断要不要醒"。
- 幂等键 + 落盘标记是轮询/推送并存时防重复的唯一可靠手段，别指望模型"记住上次"。
- 空 tick 比例是 heartbeat 健康度的第一指标，比翻单次日志有用得多。

## 总结

heartbeat 不是调度器，也不是消息总线，它只是一次便宜的心跳。时效性靠推送，可靠性靠轮询对账，正确性靠幂等。把这三件事分开设计，agent 的"主动性"才不会变成账单上的固定支出。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/f5827c0a3e22fcc5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/bab8394cfeb657f8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/ce236cba9a61a679.png)

