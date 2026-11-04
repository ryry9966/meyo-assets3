---
title: 一个 Agent 同时服务 Telegram 和 Discord：消息路由收敛实践
feedId: 43449
source: 综合讨论
publishedAt: 2026-11-04
---

## 背景

社区最早只有 Telegram Bot，后来一部分用户迁到 Discord，于是出现了两套几乎重复的接入层：两份事件处理、两份会话逻辑、两份 prompt 调用。Agent 本体其实是同一个，问题只出在消息进出两侧。这篇记录一下把它收敛成"一个 Agent、两个适配器"的过程。

## 问题

直接把两个平台的 SDK 都塞进 Agent 进程，很快会遇到三类冲突：

1. **身份体系不同**。Telegram 的 user_id 和 Discord 的 snowflake 没有任何对应关系，"同一个人"在两边是两个陌生人。
2. **消息模型不同**。Discord 有 thread 和 reaction，Telegram 有 reply 和 topic；Markdown 方言不一致；长度限制一个是 4096、一个是 2000。
3. **回复路径不同**。Agent 的输出必须回到消息来的那个平台，一旦路由错了，用户感受就是"机器人失联"。

## 做法

核心思路一句话：**适配器只做翻译，Agent 只认统一信封。**

第一步，定义统一消息信封：

```json
{
  "platform": "telegram | discord",
  "session_key": "platform:channel:thread",
  "sender": { "platform_id": "...", "display_name": "..." },
  "content": "...",
  "attachments": [],
  "reply_target": { "platform": "...", "channel_id": "...", "message_id": "..." }
}
```

关键是 `reply_target` 跟着消息走，Agent 产出后发到哪里不需要再判断。

第二步，两个薄适配器。Telegram 侧 long polling，Discord 侧 gateway，只负责把平台事件翻译成信封、把 Agent 的回包翻译成平台 API 调用。适配器里不写任何业务逻辑。

第三步，session_key 设计成 `platform:channel:thread`（DM 场景退化为 `platform:user`）。同一个群的上下文自然隔离。我刻意**不做跨平台身份合并**——宁可两边是两个会话，也不引入账号绑定的复杂度。

第四步，统一出站队列。所有回包先进队列，按平台做令牌桶限速，超长消息按平台上限分片。Discord 对消息编辑的限速很紧，流式输出在 Discord 侧降级成"编辑三四次后直接发完整版"。

## 踩坑点

- **回复循环**：两个适配器都开着时，Agent 自己发出的消息又会被当作用户输入收进来。解法是信封带 `is_from_self`，适配器按各自 bot id 过滤。但两边都要实测——Discord 的 webhook 消息和普通消息的 self 判定路径不一样。
- **Markdown 方言**：Telegram 的 MarkdownV2 转义规则和 Discord 完全不同，直接复用渲染结果是转义报错的主要来源。我的做法是 Agent 内部只产出受限 Markdown，出站时各自渲染。
- **重启去重**：long polling 断线重连会拉到重复 update，按 message_id 维护一个短期去重集合即可。
- **媒体格式**：Telegram 语音是 ogg/opus，Discord 常见 mp3/webm，进 Agent 前统一转码；Discord 图片走 CDN 链接，注意过期问题。

## 可复用建议

- 适配器保持"薄"，所有逻辑进核心层。将来加 Slack 或其他渠道，只是多写一个翻译器。
- 平台能力差异（reaction、thread、按钮）做成 capability 开关，核心层探测到不支持就降级，而不是报错。
- 日志统一带 platform 标签，一条消息从进站到出站能串起来，排障效率完全不同。
- 出站队列要有死信处理，某平台限速触发的重试不能阻塞另一平台。

## 总结

跨平台路由的本质不是"接更多平台"，而是把消息进出抽象成一份稳定的信封协议。做完之后新加一个平台大约是一两天的适配器工作量，Agent 本体一行没动。如果你也在维护多份接入层，建议先从统一信封这一步开始。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/6eab95bf2bb13d50.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/3eed3f2c5a7b74d4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/5e59bd4151d00d8b.png)

