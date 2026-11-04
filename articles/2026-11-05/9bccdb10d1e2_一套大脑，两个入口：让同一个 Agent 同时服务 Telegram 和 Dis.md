---
title: 一套大脑，两个入口：让同一个 Agent 同时服务 Telegram 和 Discord
feedId: 43500
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

社区用户一部分在 Telegram 群，一部分在 Discord 服务器。最早的做法是维护两个 Bot，各自接同一套模型 API。跑了一段时间问题就出来了：prompt 改了一边忘另一边，记忆文件存在两份，两边的回答风格逐渐分叉。这次重构的目标很简单：Agent 核心只保留一份，Telegram 和 Discord 只是两个入口适配器。

## 问题

双实例方案的痛点：

- 系统提示、工具配置、记忆目录各一份，永远同步不干净；
- 会话状态割裂，同一个人在两边问同一个问题，Agent 认为是两个人；
- 两个平台的消息格式、长度上限、速率限制完全不同，业务代码里散落着各种 `if platform ==` 分支。

## 做法

OpenClaw 侧的思路是把「收消息 → 路由 → 出站」拆成三层：

1. **统一消息信封**。两个适配器都把原始消息归一化成同一结构：`platform`、`chat_id`、`user_id`、`thread_id`、`text`、`reply_to`、`attachments`。适配器只做协议翻译，不碰业务逻辑。
2. **会话键设计**。session key 取 `(platform, chat_id, thread_id)` 的哈希。刻意把 platform 放进 key——不要试图跨平台合并同一用户的会话，身份映射是独立的权限问题，混进会话模型只会更乱。
3. **出站渲染器**。Agent 产出统一的 Markdown 草稿，由各平台 renderer 转换：Telegram 走 MarkdownV2 并处理转义，Discord 用自家方言；超过长度上限（Telegram 4096 / Discord 2000）时按段落边界分片，而不是硬切字符。

流式回复的处理：长任务先发一条占位消息，然后按 3–5 秒的节流间隔 edit 更新。两个平台对消息编辑都有限速，别每收到一个 token 就改一次。

## 踩坑点

- **MarkdownV2 转义是最大坑源**：`_`、`*`、`[` 等都要转义，模型输出代码块时最容易炸。后来改成代码块一律用 HTML `<pre>` 包裹才稳定下来。
- **分片破坏结构**：按字符数硬切会把代码块切成两半，必须按「空行 + 围栏闭合」的分段策略切。
- **Discord gateway 断线重连后可能收到重复事件**，出站侧要用 message id 做幂等去重。
- **Telegram getUpdates 的 offset 处理不对**会丢消息或重复消费；webhook 模式则要处理好 429 重试。
- **速率限制按平台各算**：Telegram 约每聊天 1 msg/s，Discord 每频道 5 请求/5 秒。群聊活跃时出站必须排队，否则两边都会被限流。

## 可复用建议

- 适配器保持「薄」。平台特有能力（发表情回应、置顶、斜杠命令）通过 MCP 工具暴露给 Agent，而不是写死在核心里，核心永远不知道自己跑在哪个平台上。
- 日志用统一 `trace_id` 从入站贯穿到出站，排查跨平台问题时能省一半时间。
- 灰度顺序：先只读（只回答、不执行动作），稳定后再开工具权限，且两个平台的白名单分开配置。

## 总结

跨平台路由的本质不是「接两个 API」，而是把协议差异压死在适配器层，让核心保持平台无关。重构完成后，评估了一下再加一个平台（比如 Slack）的工作量：一个适配器加一份 renderer，核心零改动。这套「薄适配器 + 统一信封 + 平台感知会话键」的结构，建议任何做多入口 Agent 的人在第一天就定下来，后补的代价远高于起步时多想十分钟。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/33d3f4a14fe22425.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/da49dac164a42d8e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/89b7dbc87b414628.png)

