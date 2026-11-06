---
title: cron 还是 heartbeat：OpenClaw 两种定时机制的选型与踩坑
feedId: 43628
source: 综合讨论
publishedAt: 2026-11-06
---

## 背景

OpenClaw 的 agent 常驻在一台 VPS 或家里的小机器上，除了被动响应消息，大家最常用的就是“让它自己动”：每天早上发日报、每小时抓一次数据、隔半小时巡检一下待办。平台为此给了两条路：**cron（定时任务）** 和 **heartbeat（心跳）**。两者都能周期性唤醒 agent，但设计意图完全不同，混着用是新手最常见的坑。

## 问题

我一开始就踩过：把“每天 9 点发日报”写进 HEARTBEAT.md，指望 agent 到点干活，结果它在 8:47 就发了，有时一天还发两三遍；后来又矫枉过正全用 cron，给每段巡检逻辑单开一个 job，一周后 cron 列表十几条，互相打架还重复推送。根因是没分清两种机制的触发模型。

## 本质区别

- **cron 是时间驱动**：到点触发一次独立运行，每个 job 可以指定目标 session 和 payload，跑完即止。适合“什么时候做、做什么都确定”的任务。
- **heartbeat 是状态驱动**：gateway 按固定间隔（默认 30 分钟）唤醒主会话，agent 醒来自己检查 HEARTBEAT.md、未读消息和待办，决定干点啥，还是回一句 HEARTBEAT_OK 继续睡。适合“不确定什么时候有事，但要有人看着”的场景。

## 做法

cron 典型配置（示意）：

```yaml
name: morning-digest
cron: "0 9 * * *"
session: main
message: 汇总昨天的频道消息，生成日报发到 Telegram
```

heartbeat 典型用法：在 workspace 根目录放 HEARTBEAT.md，写清楚巡检项，并明确“无事可做就返回 HEARTBEAT_OK”；间隔通过 `HEARTBEAT_INTERVAL_MS` 调整，30 分钟起步。

选型一句话：**到点要结果用 cron，没事看一眼用 heartbeat。**

## 踩坑点

1. **时区**：cron 按容器/系统时区解析，服务器是 UTC 时“9 点”会差 8 小时，先把 `TZ` 设对。
2. **心跳很贵**：每次心跳都是一次真实的模型调用，5 分钟一次等于一天 288 次。多数场景 30–60 分钟足够。
3. **cron 运行上下文孤立**：不指定 session 或每次新开会话，agent 不知道自己昨天说过什么，日报会前后矛盾。
4. **别把重活塞进心跳**：心跳共享主会话，一次抓全站数据会阻塞正常对话，重活应交给 cron。
5. **重复触发**：cron 和 heartbeat 同时盯一个任务，消息会推两遍。一条链路负责触发，另一条只负责处理。

## 可复用建议

一个稳定的组合套路：**cron 管输入，heartbeat 管收敛**。cron 定时抓数据、把结果写进状态文件；heartbeat 醒来时先看状态文件，有增量才汇总推送。这样触发与处理解耦，天然幂等，在状态文件里加个“已处理”标记就能顺便去重。

## 总结

cron 是闹钟，heartbeat 是值班员。确定性任务交给闹钟，不确定的看护交给值班员，再用一个状态文件把两者接起来——这套结构搭好之后，agent 的自动化部分基本就不会出乱子了。有别的组合玩法欢迎在评论区交流。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/0ae65a7ebe0766b9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/11df983d50aeb4ae.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/a6d6ea5ce6ba534d.png)

