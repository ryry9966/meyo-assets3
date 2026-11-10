---
title: OpenClaw 定时任务选型：cron 和 heartbeat 到底怎么分
feedId: 44052
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景

OpenClaw 里有两套都能"周期性干活"的机制，新手最容易混着用：

- **cron jobs**：标准 cron 表达式驱动的调度器，到点触发一次**独立的 agent 运行**（隔离 session），结果可投递到频道。
- **heartbeat**：网关定期向**常驻主会话**发送心跳消息，agent 醒来后读 workspace 里的 `HEARTBEAT.md` 决定做什么，没事就静默应答（`HEARTBEAT_OK`，网关会拦截不上报）。

一个像"闹钟 + 独立会议室"，一个像"定时巡逻 + 同一个大脑"。

## 问题

实际使用中反复出现的几个困惑：

- 每天 9 点把日报发到 Telegram，该用哪个？
- 想让 agent 每半小时看一眼收件箱和监控，heartbeat 会不会太重？
- 为什么 heartbeat 跑一晚上，token 账单翻倍、频道还被刷屏？
- 为什么 cron 任务里的 agent"失忆"，答非所问？

这些基本都是机制选错或配置不到位导致的。

## 做法

**1. 到点、精确、独立 → 用 cron**

```bash
openclaw cron add --name daily-report \
  --cron "0 9 * * *" \
  --message "汇总昨日 commits 生成日报，发到 telegram"
```

cron 触发的是隔离 session，**不带主会话上下文**。适合定时报告、定点提醒、定时摘要这类输入输出都明确的任务。

**2. 巡逻、跟随上下文 → 用 heartbeat**

配置里开启 heartbeat（默认间隔 30 分钟），并认真维护 `HEARTBEAT.md`：

```md
- 有未读邮件则做一句话汇总
- NAS 磁盘占用 > 85% 则告警
- 其余情况一律 HEARTBEAT_OK，不要输出
```

heartbeat 共享主会话的记忆和对话上下文，适合持续巡检、待办整理、记忆维护这类"需要知道自己是谁、最近干了什么"的任务。

**3. 组合拳**

我目前的分工：cron 负责**对外输出**（日报、周报、定点提醒），heartbeat 负责**对内维护**（收件箱巡检、记忆清理）。心跳发现问题就在主会话里记结论，cron 只管到点执行，互不抢戏。

## 踩坑点

1. **heartbeat 间隔改太短**。有人拿它做秒级监控，token 消耗和主会话占用直接爆炸。高频探测交给 cron 或外部脚本，心跳只做低频巡逻。
2. **cron 的失忆问题**。隔离 session 没有主会话记忆，message 里要自带上下文：角色设定、数据来源、输出渠道三要素写全。
3. **HEARTBEAT.md 忘写静默条款**。不写"无事不要回复"，agent 每次心跳都可能向你汇报"一切正常"，频道被刷屏。
4. **时区**。cron 按网关所在机器的时区跑，VPS 在 UTC 时，`0 9 * * *` 不是你以为的早上 9 点。
5. **主机休眠**。笔记本合盖，心跳和 cron 一起断。常驻任务建议放 NAS / 小主机 / VPS。

## 可复用建议

- 判断口诀：**要准时、要投递 → cron；要上下文、要巡逻 → heartbeat**。
- 每个任务先问两个问题：需要主会话记忆吗？需要精确到分钟吗？两个答案的组合直接决定选型。
- `HEARTBEAT.md` 当 checklist 维护：任务做完就删条目，别让它无限膨胀成第二个 memory。
- cron 的 message 模板固定三段式：角色 + 数据来源 + 输出渠道，可大幅降低失忆概率。

## 总结

cron 和 heartbeat 不是竞争关系，是两条互补的动线：cron 是精确的外部触发器，heartbeat 是带记忆的内部巡逻。选错机制的代价往往不是跑不起来，而是账单、刷屏和"失忆"这三种隐性成本——先用对，再调优。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/5d0a9e9b870a31de.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/0da6f230102bfa3b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/0ef66b1c0a498804.png)

