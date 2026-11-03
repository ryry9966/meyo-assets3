---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 43394
source: 综合讨论
publishedAt: 2026-11-04
---

## 背景

用 OpenClaw 做个人自动化，绕不开两类定时触发：**cron job**（独立定时任务）和 **heartbeat**（主会话心跳）。我见过不少人把所有事都塞进 HEARTBEAT.md，也见过全用 cron 的，跑两周就开始疼：要么 token 烧得莫名其妙，要么任务静默失败没人发现。这篇把半年使用经验整理成一份选型笔记。

## 本质区别

- **cron**：独立会话、独立调度。到点唤醒 agent，跑完 prompt 即结束，结果可以 announce 到指定渠道，也可以只留在运行记录里。不占主会话上下文，每次运行独立留痕。
- **heartbeat**：按固定间隔唤醒主会话，让 agent 读 HEARTBEAT.md 自行判断要不要干活；没事就回一个 `HEARTBEAT_ACK`，静默吞掉、不通知你。它共享主会话全部记忆。

一句话：cron 是"到点干一件具体的事"，heartbeat 是"定期自查有没有事干"。

## 做法

heartbeat 在 `openclaw.json` 里开启：

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m",
        "activeHours": { "start": "08:00", "end": "23:00" }
      }
    }
  }
}
```

HEARTBEAT.md 当"活清单"用：

```markdown
### 常驻检查
- ~/inbox 有新文件时，整理并摘要发我
- 服务器磁盘 > 85% 时提醒我（每天最多一次）
```

cron 最省事的建法是直接在聊天里让 agent 建："工作日早上 9 点，把 GitHub 通知汇总发到 Telegram 群。"管理用 `/cron list` 看任务、`/cron runs` 看历史运行，静默失败基本都能在 runs 里翻到。另外 cron 支持 next-heartbeat 唤醒模式，让任务搭下一次心跳的车，省一次独立唤醒，轻任务可以这么搭。

## 怎么选

**选 cron 的信号**：有明确时间点且与上下文无关（日报、备份提醒、订阅监控）；输出要投到固定渠道；任务重、prompt 长，不想污染主会话；需要审计留痕。

**选 heartbeat 的信号**：任务依赖主会话记忆（"接着昨天那个排查"）；说不清精确触发条件，想让 agent 自己判断要不要管；频率低、任务轻。

## 踩坑点

1. **心跳间隔太短**。每次心跳都是一次完整模型调用，5 分钟一次等于一天 288 次，绝大多数只回个 ACK。30 分钟起步，夜间用 activeHours 关掉。
2. **HEARTBEAT.md 变坟场**。做完不删，agent 每次重读旧条目，甚至重复执行。约定：要么常驻、要么标一次性，完成即删。
3. **心跳任务写太重**。它跑在主会话里，大任务会撑爆上下文、提前触发 compaction，聊天记忆跟着丢。重活一律拆给 cron。
4. **cron 时区**。表达式按 gateway 配置的时区解析（默认跟宿主机走），VPS 是 UTC 时"早上 8 点"其实是北京时间下午。建任务时显式带上时区。
5. **双保险变双打扰**。同一个检查既写 heartbeat 又建 cron，收到两条通知。留一个，删另一个。
6. **主会话忙时心跳会被跳过或延后**，别指望 heartbeat 做时间敏感的事。
7. **announce 到没人看的渠道**等于没跑。cron 挂了不会喊你，定期看 `/cron runs`。

## 可复用建议

- 分层：心跳只放"轻判断"，cron 承接"重执行"。可以只写一条心跳规则："若某条件持续满足，提醒我建个 cron 任务"，把重活交给一次性确认。
- announce 渠道固定指向"通知向"会话，别混进日常聊天窗口。
- 每月清一次 `/cron list`，弃用任务先 disable 留痕，别急着删。

## 总结

cron 和 heartbeat 不是二选一，而是分工：cron 负责确定性的时间驱动，heartbeat 负责不确定的状态驱动。我的默认配置：心跳 30 分钟 + 活跃时段限制 + HEARTBEAT.md 保持在三条以内，其余全走 cron。跑一个月后回头看看运行记录和 token 账单，比任何选型文章都诚实。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/23ed353addcbed09.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/211e7eeea95239be.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/ef96dda9c142df7a.png)

