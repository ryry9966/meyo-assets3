---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 43789
source: 综合讨论
publishedAt: 2026-11-08
---

## 背景

同一个 Agent 先后被拉进 Telegram 群和 Discord 服务器，最直接的做法是两边各跑一套实例。跑了一个月就扛不住了：提示词两处维护、记忆不互通、插件逻辑改一处漏一处。这篇记录我把它收敛成「一个 Agent 实例 + 两个通道适配器」的过程，思路对任何支持插件化消息通道的 Agent 框架都适用。

## 问题

把两边的 webhook 直接都接到 Agent 上，第一周就会撞上：

- 消息模型不同：Telegram 有编辑消息，Discord 有 thread 和 reaction；
- 身份体系不互通，同一个人两边是两个 ID；
- Markdown 方言不一致，一边渲染正常一边炸版；
- 长度限制和频控不同（Discord 单条 2000 字符、按 channel 限速，Telegram 4096、按 bot 限速）。

核心矛盾：业务逻辑想要统一的输入输出，平台层各有各的怪癖。

## 做法

**1. 定义内部消息信封。** 所有平台消息先进适配器，归一化成统一结构：

```yaml
envelope:
  platform: telegram | discord
  route_key: "{platform}:{chat_id}[:{thread_id}]"
  user_ref: 平台内用户引用
  content: 纯文本正文
  attachments: [引用 URL]
  event: message | edit | delete
```

**2. 适配器只管收发。** Telegram 侧处理 long polling 和 MarkdownV2 转义；Discord 侧处理 gateway 事件、thread 归属和 embed 渲染。适配器里不写任何业务判断。

**3. 路由与会话。** `route_key` 直接作为会话 key，两平台默认隔离；确需合并时在身份映射表里显式绑定，不做自动猜测。

**4. 出站交给统一 dispatcher。** Agent 输出携带路由元数据，dispatcher 按 platform 选择切分策略（在代码块围栏边界切）、parse mode，并走各自独立的限速队列。

## 踩坑点

- **回声循环**：桥接场景下 Agent 会把自己转发的消息当新输入。解决：出站消息记录 message_id，入站过滤自家 ID；对 edit 事件只更新上下文、不触发回复。
- **炸版**：Discord 的 Markdown 和 Telegram 的 MarkdownV2 差异比想象大，尤其下划线和转义。最省事的是两边都降级为「代码块 + 纯文本」，确实需要 embed 再按平台单独渲染。
- **盲切截断代码块**：按硬限长切分会把代码块腰斩，必须按围栏配对切分。
- **限速不对称**：队列必须按平台分开，各自实现退避，否则一个平台的 429 会拖垮另一个。

## 可复用建议

- 适配器接口收敛到五个方法：`send / edit / delete / react / fetch-history`，新增平台就是再写一个适配器，Agent 侧零改动。
- 信封带 schema 版本号，为后续演进留余地。
- 灰度顺序：先只读（记录不回复），再小范围回复，最后放开，别一步到位。
- 日志始终带 `route_key` 和 `event` 字段，排障时按路由过滤比按时间翻日志快得多。

## 总结

跨平台这件事，难点不在「接上」，而在把平台差异关进适配器、把路由策略显式化。统一信封 + 最小适配器接口 + 分平台限速，三件套做完后，后来接入第三个平台大概只花了一个下午。另外一个经验：先跑通隔离会话，再考虑跨平台身份合并，顺序反过来会非常痛苦。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/4b951dae9c3875b5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/3385eb023edcd0f2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/e02b4356bb31fe42.png)

