---
title: 一个 Agent，两条通道：Telegram + Discord 跨平台消息路由实践
feedId: 43877
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

我们团队日常沟通一半在 Telegram（值班群、私聊问答），一半在 Discord（项目社区、线程讨论）。之前两边各跑一个 bot，背后各挂一份 agent 配置，结果显而易见：记忆不通、prompt 维护两份、每加一个工具都要在两处同步。这次把两条通道收敛到一个 OpenClaw agent 上，让路由层负责平台适配，agent 本身只面对统一的内部消息协议。

## 问题

难点不在"多接一个 bot"，而在三层差异：

- **传输层**：Telegram 长轮询/Webhook vs Discord Gateway WebSocket；
- **消息语义**：Discord 有 thread 和 embed，Telegram 有 forum topic 和 reply，粒度对不上；
- **格式**：Markdown 方言、长度限制、转义规则各不相同。

如果把这些差异直接漏进 agent，等于让模型替你做平台兼容，既贵又不稳定。

## 做法

1. **先定统一消息 schema**：`platform / chat_id / thread_id / user_id / text / attachments / reply_to`。这是整个方案的锚点，schema 不稳，后面全白做。
2. **两个薄适配器作为 OpenClaw 插件**：Telegram 用 bot token + 长轮询（本地开发友好），Discord 用 gateway 连接。适配器只做收发和格式转换，不做业务判断。
3. **路由层生成会话键**：`platform:chat_id:thread_id` 三段式，映射到 agent 的 session。同群/同线程上下文连续，不同群互相隔离。
4. **出站格式化**：agent 输出的 Markdown 统一转成 Telegram HTML parse mode 和 Discord 普通消息/embed，超长自动分段。
5. **异步占位**：长耗时 MCP 工具调用先回一条"处理中"，完成后编辑消息或补发结果，避免用户干等。

## 踩坑点

- Telegram 的 MarkdownV2 转义几乎不可维护，果断换 HTML parse mode，省掉一大类 bug。
- Discord 有 2000 字符上限和频道级 rate limit，分段发送要按速率排队，否则会被 429 连续打脸。
- 不要强行把 Discord thread 和 Telegram topic 一一对应，语义不同，会话键里各带各的 thread_id 就够了。
- 适配器写错会话键会出现"串台"——A 群的问题在 B 群被回答。上线前务必双端同时压测。
- 附件别直接透传：Telegram 的 file_id 和 Discord CDN 链接生命周期不同，转存统一对象存储再引用更稳。

## 可复用建议

- **适配器薄、路由厚**：所有平台差异在 schema 和格式化两处消化，别散落在业务逻辑里。
- 三段式会话键设计好之后，以后接 Slack、飞书只是加一个适配器，路由层不动。
- 给每个平台留 feature flag：某些 MCP 工具只在私聊或管理频道开放。
- 出站统一走队列，重试和幂等在队列层做，不要在各适配器里各写一套。

## 总结

跨平台路由的本质不是"多接一个 bot"，而是把传输差异挡在 agent 之外。统一 schema + 薄适配器 + 三段式会话键这套结构，落地后两周路由层没再改过，改的只有 prompt 和工具配置。如果你现在也在两个平台各养一份 agent，建议优先做合并——实际成本比想象中低得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/39fc97fffa662901.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/5d31b37645929bcd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/f2466945b37fb0b0.png)

