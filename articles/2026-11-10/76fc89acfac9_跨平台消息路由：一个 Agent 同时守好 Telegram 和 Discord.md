---
title: 跨平台消息路由：一个 Agent 同时守好 Telegram 和 Discord
feedId: 44034
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景

Agent 跑起来之后，用户不会全在一个平台。团队一部分人在 Telegram 群里问，另一部分在 Discord 服务器里 @。最初的做法是部署两个 bot、两套 handler、两份 prompt，改一次工具调用逻辑要同步两处，很快就撑不住了。这次的目标很简单：一套 Agent 核心，两个入口。

## 问题

真正的问题不是“再写一个 bot”，而是三个结构性差异：

1. 消息形态不同：Telegram 的 MarkdownV2/HTML 转义规则与 Discord 的 Markdown/Embed 完全对不上；
2. 限制不同：Telegram 单条 4096 字符、每聊天约 1 条/秒；Discord 2000 字符、每频道 5 条/5 秒；
3. 会话语义不同：Discord 有 thread，Telegram 群有 topic，`reply_to` 的含义也不一样。

把这些差异直接塞进 Agent 核心去判断，代码会迅速腐烂。

## 做法

方案是把差异全部压到连接器层（connector），核心只认一种内部信封（envelope）：

1. **定义统一信封**：`platform / user_id / channel_id / thread_id / text / attachments / reply_to / capabilities`。其中 `capabilities` 声明该渠道支持什么（embed、图片等）。
2. **两个薄连接器**：Telegram 走 Bot API 长轮询，Discord 走 gateway websocket，只做收发和格式翻译，不带任何业务逻辑。
3. **共享会话存储**：session key 用 `platform:channel_id`（群聊）或 `platform:user_id`（私聊），平台之间默认上下文隔离，避免串台。
4. **出站统一排队**：所有回复先进队列，发送器按目标平台限速规则分发；超长消息先按平台限制切分，再按能力降级（不支持 embed 就转纯文本）。
5. **工具只注册一次**：平台特有能力通过 capabilities 判断，不支持就静默走通用路径。

## 踩坑点

- **Markdown 转义最容易翻车**。MarkdownV2 下 `_`、`*`、`[` 都要转义，直接把 Discord 的回复丢过去会报 400。最终我们统一用内部中间格式，出站时各自渲染。
- **重试导致重复回复**：网络抖动后连接器重发，Agent 会再答一次。给每条消息加 dedupe key，入站去重。
- **断线漏消息**：Discord gateway resume 失败后要有全量拉取兜底；Telegram 注意 getUpdates 的 offset 提交时机。
- **并发竞态**：两个连接器写同一个会话存储时出过脏数据，按 session key 加了简单锁才稳住。
- **身份别乱合并**：user ID 只在渠道内唯一，跨平台身份合并涉及隐私，除非明确需要不做。

## 可复用建议

1. 连接器保持“薄”，只做翻译，业务逻辑永远在 Agent 层；
2. 用能力声明（capabilities）代替 if-else 判断平台名，可维护性高得多；
3. 出站统一过限速队列，切分、降级都在这一层解决；
4. 日志带 `platform:channel_id` 前缀的关联 ID，排障时一眼定位；
5. 上线前用 dry-run 模式同时灌两端假消息做回归。

## 总结

跨平台路由的本质不是“多接一个平台”，而是把平台差异隔离在边界上，让核心保持无知。改造之后，接入第三个渠道预计只需一个新的薄连接器和一份限速配置，Agent 核心零改动。“薄连接器 + 统一信封 + 能力协商”这组结构，建议在第一个渠道接入时就搭好，而不是等第二个平台出现再重构。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/a5d0d9148440886d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/4eed12e0ccb539b9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/c35314ff24b3070f.png)

