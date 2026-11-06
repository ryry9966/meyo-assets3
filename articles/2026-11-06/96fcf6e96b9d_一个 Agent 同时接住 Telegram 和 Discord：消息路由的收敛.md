---
title: 一个 Agent 同时接住 Telegram 和 Discord：消息路由的收敛实践
feedId: 43659
source: 综合讨论
publishedAt: 2026-11-06
---

## 背景

社区用户一部分在 Telegram，一部分在 Discord。最初的做法是部署两个 Bot，各自挂一套 prompt 和工具。跑了一阵子问题就出来了：两边记忆不同步、人格设定漂移，修一个 bug 要改两处配置。这篇文章记录我们把两条链路收敛到一个 Agent 上的过程，核心是在中间加了一层统一的消息路由。

## 问题

把「接消息」和「跑 Agent」拆开后，跨平台真正剩下三个难点：

1. **会话归属**：同一个用户在两个平台找 Agent，算两个会话还是同一个？
2. **能力差异**：消息长度上限（Telegram 4096 / Discord 2000）、Markdown 方言、话题结构（Discord thread vs Telegram topic）都不一样。
3. **限流与重试**：两个平台的限流维度完全不同，重试参数没法共用一套。

## 做法

OpenClaw 侧的结构大致是：

```
Telegram adapter ─┐
                  ├─► Router ─► Session Store ─► Agent
Discord adapter ──┘                  │
                                     ▼
                          Reply Renderer（按平台回染）
```

关键决策三个：

**1. 入站统一，出站分化。** 两个 adapter 只负责把平台消息转成同一个内部信封（sender、platform、chat_ref、content、attachments），Router 和 Agent 只见信封。回复时由 Reply Renderer 按目标平台拆长文、转义 Markdown、处理 @。

**2. 会话键用 `platform:user_id`，共享层单独抽。** 我们没有做跨平台身份自动合并——默认同一个人在两个平台是两个会话，但共享同一份长期记忆（通过 MCP 挂的 memory server）。确需合并的用户走一张手工维护的映射表。自动合并听起来美好，实测误合并一次就很难收场。

**3. 限流参数留在 adapter 里。** Router 只管排队和幂等，每个 adapter 自己清楚平台节奏：Discord 按 channel 约 5 条/5 秒，Telegram 每聊天约 1 条/秒，退避策略各自配置。

## 踩坑点

- **Telegram 409 冲突**：本地用 polling、线上用 webhook，同一个 token 两边跑会互相踢连接。测试环境务必换独立 token。
- **Markdown 转义**：Telegram MarkdownV2 比 Discord 严格得多，`_` `*` `[` `]` 都要处理。我们的解法是 Agent 输出统一收敛到受限 Markdown 子集，Renderer 再各自翻译。
- **长消息拆分**：按 token 拆会切断代码块。后来改成按「段落 + 代码块边界」拆，代码块超限时降级成文件附件（Discord 直接传，Telegram 走 sendDocument）。
- **Discord thread 回复**：不显式透传 thread_id 时，消息会掉到主频道。这个问题排查了很久，最后发现是 adapter 上下文没带全。

## 可复用建议

- 越早统一内部消息模型越好，后补的兼容层只会越积越多。
- 平台差异全部关进 adapter，Router 和 Agent 保持平台无关；加第三个平台（比如 Slack）理论上只需新写一个 adapter。
- 每条消息日志带上 platform 和 chat_ref，跨平台排障时这是唯一的定位手段。
- 幂等键用 `platform:chat_ref:msg_id`，重试前先查去重表，能省掉不少重复回复。

## 总结

跨平台的难点不在「多接一个 Bot」，而在路由层的抽象质量。入站统一、出站分化、身份谨慎合并，这三条做到位后，一个 Agent 服务多个平台是稳的。两个 adapter 加路由层大概一天能跑通，但转义和拆分的细节建议预留两天调试，别低估这两个长尾。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/1590ccf31cea8747.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/672769378b386646.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/812a3056c0d620e5.png)

