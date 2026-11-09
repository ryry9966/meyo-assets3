---
title: 跨平台消息路由：一个 Agent 同时值守 Telegram 和 Discord
feedId: 43964
source: 综合讨论
publishedAt: 2026-11-10
---

# 背景

过去半年，我的 Agent 挂在 Telegram 上处理私聊、定时提醒和临时查询；社区协作和讨论则都在 Discord。最初的做法是再起一个实例接 Discord，两周后就出问题了：两份 workspace 的记忆开始分叉，同一个问题两边答案不一致，工具和 MCP 配置改一处漏一处。最终收敛成现在的方案：**一个 OpenClaw Agent、一个 gateway、两条 channel 入口**。本文记录关键决策和踩过的坑。

# 问题：连通之后才是开始

OpenClaw 的 gateway 原生支持多 channel，把 token 填进去消息就能收发，"连通"本身不难。真正的问题在语义层：

1. **会话隔离**：Telegram 私聊、Discord 频道、Discord 私聊必须是独立 session，否则上下文串台；
2. **身份分裂**：同一个人在两个平台是两个 ID，Agent 记忆里会长出"两个人"；
3. **格式差异**：Telegram 的 parse_mode 对 Markdown 很挑剔，Discord 有自己的引用和 thread 习惯，硬编码一种格式必翻车。

# 做法

**1. 单 agent 多 channel，不要双实例。** 核心就是让两个 channel 指向同一个 agent id（字段名以你的版本为准，示意）：

```json5
{
  agents: [{ id: "main", workspace: "~/agent-main" }],
  bindings: [
    { agent: "main", match: { channel: "telegram" } },
    { agent: "main", match: { channel: "discord", guildId: "..." } }
  ]
}
```

共用 workspace 意味着记忆、工具、cron 都只有一份，改配置不会再漏。

**2. 会话键交给框架，别自己合并。** session 按 channel + peer 隔离是特性不是 bug。确有"跨平台同人"需求，用 pairing/identity 机制手动绑定两个 ID，不要写自动匹配——误绑的清理成本远高于手动维护。

**3. 出站格式化放在适配层。** 约定 Agent 只输出中性文本，发送前按 channel 渲染：Discord 走 embed 和 thread 回复；Telegram 用 MarkdownV2 发送，失败降级纯文本重发一次。这一条约定解决了我大部分发送报错。

**4. 群聊触发收紧。** Discord 只在被 @ 或白名单频道响应；Telegram 群要求被提及或触发词。否则 Agent 会礼貌地回复每一句寒暄。

**5. 幂等去重。** 两个平台都可能重推事件，按消息 ID 做去重窗口。重复回复比不回复更伤信任。

# 踩坑点

- **Discord 收到空消息**：没开 MESSAGE CONTENT INTENT 时 content 全为空，表现为"在线但不回话"，非常迷惑。这是第一个要查的地方。
- **Telegram 静默发送失败**：MarkdownV2 里未转义的 `_`、`(` 会让整条消息发不出去，部分库只打 warning。降级重发必须有。
- **双 gateway 复用同一 token**：测试期起过第二个进程，两个进程同时消费 update，出现重复回复和消息漂移。一个 token 只允许一个进程持有。
- **Discord 限速**：单频道约 5 条/5 秒，批量通知不排队就是 429。
- **平台风格写进全局 prompt**：Agent 在 Discord 里学会了滥用 @。风格提示按 channel 注入，不要进 system prompt。

# 可复用的约定

- 定一个 canonical 消息模型：inbound 归一化、outbound 渲染，Agent 永远只见中性结构。以后接 Slack 只是加一个适配器。
- 日志和指标都带 channel 标签，出问题第一眼就能定位是哪条腿瘸了。
- 灰度顺序：先在 Discord 测试服跑一周，再放开 Telegram 私聊——私聊错发的代价更高。
- 工具与 MCP 配置只挂在 agent 上，channel 层不复制任何业务逻辑。

# 总结

多平台不是"多接一个 channel"的搬运活，而是把传输、路由、身份、格式四层拆开：前两层 OpenClaw 基本替你做了，后两层要靠你自己定约定并守住。守住之后，一个 Agent 值守多个平台是稳定且省心的——我目前的增量成本，基本只剩"新平台加一个适配器"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/e56a802b5d7f5e97.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/4572abe9e8b458a0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/bf141140feabfdcc.png)

