---
title: 一个 Agent，两个入口：让 OpenClaw 同时服务 Telegram 和 Discord
feedId: 44121
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景

我最早是在 Telegram 上跑通了个人 Agent：日常问答、备忘、定时摘要都在里面。后来团队协作迁到了 Discord，需求很自然——在 Discord 里 @ 同一个 Agent，它得认识我，记得之前聊过什么，工具权限也一致。第一反应是再起一个实例，跑了一周就放弃了。

## 问题

- **两份实例 = 两份人生**：配置重复维护，记忆分裂，同一件事要在两个平台各问一遍；
- **平台差异不小**：Discord 单条 2000 字符、Telegram 4096，Markdown 方言不同，群组里"Bot 能看见什么"的规则也不一样；
- **会话策略要早定**：同一个人出现在两个平台，算一个会话还是两个？

## 做法

1. **单网关、多通道**。OpenClaw 的 gateway 是中心进程，通道只是适配器。配置里同时启用两个通道，各填 token（走环境变量）：

```json
{
  "channels": {
    "telegram": { "botToken": "${TELEGRAM_BOT_TOKEN}" },
    "discord":  { "botToken": "${DISCORD_BOT_TOKEN}" }
  }
}
```

2. **两个通道绑定同一个 agent / workspace**，这是共享记忆的关键。不要起两个进程再想办法"同步"。
3. **会话键用 (channel, chat, user)**，而不是全局单会话。两个平台的上下文天然隔离，避免互相污染；确实需要跨平台连续的任务，放进共享 workspace 文件里解决。
4. **输出适配**：约束 agent 默认输出简洁 Markdown；超长回复由网关按段落/代码块边界分片，不做字符硬切。
5. **权限分通道收敛**：Telegram 是私聊白名单，全工具开放；Discord 按 server/channel 设 allowlist，shell、写文件这类高危工具只对私聊启用。
6. **灰度上线**：Discord 先只开只读问答，观察一周日志，再放开定时任务和写操作。

## 踩坑点

- **Discord 收到消息但 content 为空**：开发者后台没开 Message Content Intent。Bot 在线、事件到达、内容全空，第一反应往往是"prompt 写错了"，其实一句话的事。
- **Telegram 群组没反应**：Bot 默认 privacy mode 只能收到 /命令和显式回复。要么关掉 privacy mode，要么让用户回复它的消息。
- **Discord 2000 字符上限**：分片必须按语义边界切，硬切会把代码块劈成两截，另一半直接语法报错。
- **共享 workspace 的双刃剑**：一边的闲聊会把另一边的任务上下文挤出去，session 压缩策略要盯着 token 占用。
- **触发方式不对称**：Discord 靠 mention、Telegram 私聊全量进入。测试时"没反应"往往只是没 @ 到，日志先看"收到没有"，再看"收到之后做了什么"。

## 可复用建议

- 把通道当"插头"而不是"实例"：权限、格式、会话策略全部收敛在 agent 侧配置，通道层只做传输和适配；
- 每条消息日志打上 channel 标签，排障时先区分"没收到"和"收到没回"；
- 每个通道做心跳监控，掉线主动告警，别等用户来报障；
- 新通道上线节奏固定：限权限 → 限人群 → 看日志 → 放开。

## 总结

跨平台不是跑两份服务，而是"一个大脑，多个入口"。网关解决传输，真正要想清楚的是三件事：会话怎么隔离、输出怎么适配、权限怎么收敛。把这三件事做成配置而不是代码，之后再加 Slack 或其他通道，也只是多一段通道配置的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/6753cfd5639a45b6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/6952e2cd6a5569bd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/ee346f480ac4df47.png)

