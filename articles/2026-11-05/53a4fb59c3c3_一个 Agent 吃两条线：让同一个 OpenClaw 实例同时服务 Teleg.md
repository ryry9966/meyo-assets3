---
title: 一个 Agent 吃两条线：让同一个 OpenClaw 实例同时服务 Telegram 和 Discord
feedId: 43496
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

单 agent 单渠道是大多数人的起点：OpenClaw 挂一个 Telegram，手机上随手问。但桌面场景和社区场景天然在 Discord，于是有人干脆跑两个实例。问题随之而来：记忆分裂成两份、system prompt 和插件配置双份维护、同一个任务在两边各问一遍。

这篇记录把两个渠道收敛到一个 agent 进程上的做法。结论先行：这是路由问题，不是智能问题。

## 问题拆解

三件事必须想清楚：

1. **路由**：消息从哪个渠道进来，绑到哪个 session；
2. **格式**：同一份回复怎么适配两套渲染规则；
3. **身份**：同一个人在两个平台 ID 不同，记忆要不要合并。

## 做法

**1. 单 gateway 挂双 channel。** 配置里同时启用 telegram 和 discord，指向同一个 agent。注意 token 只能被一个进程消费——Telegram 的 getUpdates 有排他性，两个实例抢同一个 token 会直接 409 冲突。这是最高频的翻车点。

**2. session 键用 `channel:chatId`。** 默认每个（渠道， 会话）独立 session，互不污染。不要图省事把所有消息并成一个 session——两个群同时说话会串台，这是新手最容易犯的错。

**3. 身份合并靠 memory，不靠硬路由。** 自建 identity 映射表工程量不小。更务实的做法：在 DM 里让 agent 把“我的 Telegram 身份是某某”写进 memory，两个平台的上下文自然对齐。够用，且可随时修正。

**4. 输出平台中立。** system prompt 里约定 agent 只输出标准 markdown，格式转换全部下沉到各渠道适配层；同时注入当前渠道名，让 agent 知道自己在哪说话（Discord 可以更口语，TG 私聊可以更简短）。

**5. 白名单先行。** allowlist 先只放自己的 TG chat id 和 Discord 用户 ID，跑一周稳定后再开放群聊。

## 踩坑点

- Discord bot 必须在开发者后台开 **Message Content Intent**，否则收到的消息内容为空，表现为“bot 装死”；
- Discord 单条 2000 字符上限，长回复要按段落边界切分，绝不能从代码块中间硬切；
- Telegram MarkdownV2 的转义是重灾区，适配层支持的话优先选 HTML parse mode；
- 一台 VPS + 本地各跑一份是最常见的 409 来源，同一个 token 全局只能一份；
- 心跳/长任务类通知会在两个渠道同时弹出，需要为通知指定渠道或在 prompt 里限制。

## 可复用建议

- **session 模型先于一切**：先定 session key，再谈格式和媒体；
- agent 只管语义，渠道差异全部下沉适配层，不要让模型输出平台特有语法；
- 排障日志带渠道前缀（`[tg]` / `[dc]`），定位问题快一个量级；
- 语音、图片等多模态消息先只在一端开，稳定后再开另一端，避免变量叠加。

## 总结

一个 agent、多个入口、记忆单份，是比多 agent 互联便宜得多的形态。路由模型定对了，剩下的都是适配层的脏活——脏活集中在适配层，恰恰是架构做对了的标志。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/d67e78158224a7f3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/8ccbcc8ca735d9f7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/9efaec316d41cc6e.png)

