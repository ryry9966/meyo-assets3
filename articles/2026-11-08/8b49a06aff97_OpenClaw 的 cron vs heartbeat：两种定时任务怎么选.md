---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 43770
source: 综合讨论
publishedAt: 2026-11-08
---

## 背景

OpenClaw 里有两种容易混淆的"定时"机制：**heartbeat**（心跳）和 **cron**（计划任务）。heartbeat 让 agent 按固定间隔自唤醒一轮，读工作区里的 `HEARTBEAT.md` 决定"这次要不要干活"，无事时简短应答即可；cron 则是传统计划任务，按 cron 表达式在指定时刻把一段 prompt 交给 agent 执行，支持隔离会话和结果投递。两者都能"定时"，但设计意图完全不同。

## 问题

社区里常见的两类翻车：

1. 用 heartbeat 做"每天 8:30 发晨报"，结果时间飘到 9 点多——heartbeat 是**间隔驱动**，不对齐挂钟时间。
2. 用 cron 做"每 5 分钟巡检一次"，一天烧掉的 token 比聊天还多——cron 每次都是完整的 agent 轮次，高频下成本失控。

本质是：定时 ≠ 周期，周期 ≠ 定时。

## 怎么选、怎么做

一句话判据：**要精确钟点，用 cron；要"每隔一段时间看一眼"，用 heartbeat。**

### cron 的典型场景与步骤

适合每日/每周定点报告、一次性提醒、定时触发外部脚本。

- 通过 `openclaw cron` 子命令添加，或在对话里让 agent 调用 cron 工具；
- 配置四件事：**表达式**（如 `30 8 * * *`）、**payload**（一段自包含的 prompt，写清要做什么、输出什么格式）、**session**（用隔离会话保证上下文干净）、**delivery**（指定投递渠道和目标，否则产出没人看得到）。

### heartbeat 的典型场景与步骤

适合环境巡检：有没有新邮件、后台任务跑完没、磁盘是否告警。

- 把检查项写进 `HEARTBEAT.md`，保持清单式、每行一条；
- 明确写上"无事可做时简短应答即可"，避免每轮长篇输出；
- 需要更省就调低频率（默认 30 分钟，配置项 `agents.defaults.heartbeat.every`）。

### 组合用法（推荐）

heartbeat 当"看门人"：巡检发现异常（如构建失败）就写入待办；cron 管"定点动作"：每天 18:00 汇总当天待办投递到群里。感知归 heartbeat，触发归 cron。

## 踩坑点

1. **heartbeat 不补跑**。机器休眠、进程掉线错过的轮次就没了，别指望它做可靠告警。
2. **heartbeat 跑在主会话里**，产出会混进日常对话上下文，所以务必要求它输出简短，别把长报告交给心跳轮。
3. **cron 隔离会话没有聊天上下文**。别在 payload 里写"接着刚才的话题"，需要的上下文直接贴进 prompt。
4. **时区**。Docker 部署下容器常是 UTC，`0 9 * * *` 实际是北京时间 17 点，要么改容器时区，要么按 UTC 写表达式。
5. **`HEARTBEAT.md` 越写越长**。它每轮都要读，几百行清单等于每 30 分钟白烧一遍 token，定期清理。
6. **高频 cron 自我重叠**。任务运行时间超过间隔时下一轮会叠上来，脚本类任务加锁或拉长间隔。

## 可复用建议

- **判定树**：输出要投递给人、要精确时点 → cron；只影响 agent 自身状态、低频观察 → heartbeat；两者都行 → 默认 cron，语义更清楚。
- payload 和 `HEARTBEAT.md` 都当配置对待：自包含、简短、写清退出条件。
- 确定性工作下沉到脚本和工具，agent 只做编排和判断，省 token 也省心。
- 上线前用 `openclaw cron list` 核对任务列表和最近执行记录，确认时区、间隔、投递三件事都正确。

## 总结

cron 和 heartbeat 不是竞争关系，而是两个粒度：**cron 管"钟点"，heartbeat 管"节律"**。把定时触发交给 cron，把环境感知交给 heartbeat，并让各自的输出落在正确的会话和渠道里，大部分"定时任务不灵"的问题就消失了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/ce2cc7b1f1523063.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/3cb7588aca61ce52.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/ff9e4c7a04eb2785.png)

