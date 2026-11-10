---
title: 一个大脑，多张嘴：同一个 Agent 同时服务 Telegram 与 Discord 的路由实践
feedId: 44003
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景

社区用户一半在 Telegram，一半在 Discord。最初的方案很朴素：各起一个 bot，各自配 prompt、各自的会话存储。跑了一个月问题就出来了——同一个人在两边问同一个问题，得到两个不同答案；agent 记忆完全割裂；改一处配置忘改另一处，行为开始漂移。

## 问题

拆开看是四类：

1. **状态割裂**：两套 session，记忆不互通。
2. **格式差异**：Telegram 的 MarkdownV2 与 Discord 的 markdown 方言不同，长度上限一个 4096 一个 2000，Discord 还有 embed。
3. **触发差异**：Discord 靠 mention/slash，Telegram 靠命令或回复。
4. **限流差异**：两平台的频控规则和粒度都不一样。

## 做法

核心思路：**渠道只是传输层，agent 核心保持平台无关**。

1. **统一消息信封**。两个 channel 插件收到消息后不直接喂模型，先归一化成统一结构：`platform / chat_id / user_id / thread_id / text / attachments / reply_to`。模型和工具只看得见信封。
2. **会话键带平台前缀**。`telegram:dm:{user_id}`、`discord:guild:{gid}:chan:{cid}`。别用裸数字 ID，两平台的 ID 空间毫无关系但长得一样。
3. **身份打通（可选）**。一张 user_map 表加显式绑定命令，把两边账号关联，记忆合并到同一 persona。坚决不做自动猜测关联。
4. **出站渲染层**。统一回复先过 per-platform formatter：Telegram 侧用 HTML parse mode，Discord 侧做 2000 字符分块，embed 只留给结构化结果。
5. **独立限流**。每平台一个令牌桶，出站走串行队列。Telegram 群组按分钟配额，Discord 按 channel 粒度，计数互不共享。

## 踩坑点

- **MarkdownV2 转义是无底洞**。切到 HTML parse mode 才算消停；富文本一律在适配器层生成，别让模型直接吐平台格式。
- **流式输出撞频控**。Discord 靠反复 edit 消息模拟打字机效果，编辑太密会被限流。改成 800ms 节流，短回复干脆整段发。
- **thread 语义不要强行对齐**。Discord 的 thread 和 Telegram 的 forum topic 是两回事，各自保留原值即可，别抽象成同一种“子会话”。
- **会话串台**。上线第一周就遇到两平台数字 ID 撞车、上下文串掉的事故。教训：任何键里永远带来源前缀。
- **附件格式**。Telegram 语音是 ogg/opus，Discord 常见 mp3/m4a，转码放在适配器做，agent 不该关心容器格式。

## 可复用建议

- 先用纯文本把全链路打通，再补富格式。一上来就磨 embed 会拖死进度。
- 信封字段宁可冗余不可缺失；日志带 correlation id，排障时能沿一个 ID 串起收发两端。
- 适配器里平台相关的行为，用真实 payload 做快照测试，文档和实际行为经常对不上。
- 身份合并必须显式确认。自动关联只要错一次，用户就不会再信这个 agent。

## 总结

这件事的本质是“一个大脑、多张嘴”。难点不在 agent 本身，而在边缘：格式、限流、身份、会话键。把复杂度全部压进适配器，核心保持平台无关，之后要接 Slack 或其他渠道，就是再多写一个 adapter 的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/e00af4dc67fd0110.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/5f8a1b2f14e5b8e7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/e848c3d4a422b7b1.png)

