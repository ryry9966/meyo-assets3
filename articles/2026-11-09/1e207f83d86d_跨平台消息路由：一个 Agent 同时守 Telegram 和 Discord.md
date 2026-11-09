---
title: 跨平台消息路由：一个 Agent 同时守 Telegram 和 Discord
feedId: 43883
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

我最早只给 Agent 挂了一个 Telegram bot，跑在 OpenClaw 网关上，主要是自己用。后来小组想把同一套能力——日常问答、定时任务、MCP 工具调用——搬进 Discord 服务器。第一反应是再起一个实例，跑了两周就发现不对：两份配置各自漂移，记忆和会话完全割裂，同一个问题两边答案不一致，排障还得翻两份日志。

## 问题

需求其实很朴素：**一个 Agent 实例，两个入口，会话互不污染，工具与插件共享**。核心矛盾在“路由”——消息从不同平台进来，格式、身份体系、接口限制都不一样，不能让每个 adapter 各自直连模型了事。

## 做法

1. **单实例双 adapter**。Telegram 走 long polling，Discord 走 gateway websocket，都指向同一个网关进程。token 放环境变量，配置文件里只声明“哪些聊天接入、用什么会话键”。
2. **统一会话键**。我用 `channel:chat_id` 作为 session key，比如 `telegram:12345`、`discord:guild#channel`。两个平台天然隔离，避免串台。不要急着做“跨平台身份合并”，那是另一个独立的大坑。
3. **入站归一化**。adapter 层只做一件事：把 mention、回复引用、附件转成统一的内部消息结构。Telegram 的 HTML 实体和 Discord 的 markdown 方言都在这一层消化掉，Agent 拿到的永远是干净文本。
4. **出站分型**。回复时按目标渠道选格式化器和分段器：Telegram 单条 4096 字符，Discord 2000，超长内容统一走分段发送加文件兜底。
5. **权限与会话内串行**。每个会话维护一条串行队列，避免同一会话被两个渠道的并发消息打穿；渠道级 allowlist 控制谁能触发哪些技能。

## 踩坑点

- Discord 必须在开发者后台开 **Message Content Intent**，否则收不到正文，新手最容易在这里卡一小时。
- 起了两个进程同时 poll 同一个 Telegram token，会互相抢 update，表现为“消息随机丢失”。用容器或 systemd 做单实例约束，并在配置里写死。
- Discord 的速率限制比想象中严，连发分段消息容易被 429，出站要做退避重试。
- 表情回应触发、线程回复（Discord thread 对 Telegram reply_to）语义不同，别指望一层 if 搞定，老老实实在 adapter 里各写各的映射。
- 语音输入：Telegram 有 voice，Discord 用户习惯发附件，Agent 侧要统一抽象成“音频输入”，否则技能判断会漏。

## 可复用建议

- **adapter 保持“薄”**，只做协议翻译；路由规则用声明式配置，别写死在代码里。
- 日志统一打上 `channel:` 前缀标签，排障时一眼分辨流量来源。
- 用录制消息做回放测试，改路由规则前先跑一遍 fixture，比在线上试错便宜得多。

## 总结

多平台接入的难点不在“连上”，而在会话模型和格式差异的消化层。把归一化下沉到 adapter、把路由收敛到声明式配置之后，后来我再加一个渠道只花了半天。一个实例、一份记忆、多入口，是我目前验证过最省心的形态。欢迎在评论里交流你们的 session key 设计。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/b066308f40757690.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/36734bd7d0e99af0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/58c495e088fce328.png)

