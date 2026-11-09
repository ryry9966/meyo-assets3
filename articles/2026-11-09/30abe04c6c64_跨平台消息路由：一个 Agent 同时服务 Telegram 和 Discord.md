---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord 的实践
feedId: 43893
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

我的 Agent 最初只在 Telegram 上服务自己。后来有了 Discord 的需求（社区群答疑），第一版的偷懒做法是再起一个进程、各自挂一份配置。跑了一个月问题就出来了：两边会话历史完全独立，上午在 Telegram 交代的上下文，下午在 Discord 里它一无所知；提示词改了一边忘另一边；token 账单近乎翻倍。结论很直接：**大脑只能有一份，通道可以有很多条。**

## 问题拆解

“多接一个平台”真正要解决的是三件事，而不是多填一个 bot token：

1. **会话归属**：谁和谁的上下文算一条线；
2. **出站适配**：两家的 Markdown 方言、长度上限、回复/线程语义互不兼容；
3. **事件归一**：不同平台原始事件结构差异很大，不归一就直接喂模型，提示词会迅速膨胀。

## 做法与步骤

分三层，每层只干一件事：

- **归一层**：把所有入站消息转成统一信封：`platform / chat_id / user_id / thread_id / text / attachments / reply_to`。
- **路由层**：session key 取 `{platform}:{chat_id}:{thread_id}`。刻意不做跨平台共享会话，避免两边上下文互相污染；但 memory store 和工具状态全局共享，长期记忆是打通的。
- **适配层**：出站按通道走各自的 formatter。Telegram 用 HTML parse_mode（躲开 MarkdownV2 的转义地狱），Discord 用原生 markdown；长消息按 4096/2000 上限切分，且优先在代码块边界断开。

触发规则也放在路由层：Telegram 私聊全响应、群聊仅响应 @；Discord 仅响应 mention 与所在 thread 内回复；再配一个 allowlist，防止被拉进陌生群后失控。

```yaml
routing:
  session_key: "{platform}:{chat_id}:{thread_id}"
  formatter: per_channel
  allowlist: [chat_id_a, chat_id_b]
```

## 踩坑点

- Discord 不开 MESSAGE CONTENT intent 时，事件收得到但正文是空的，我排查了一小时才想起来这一项。
- 两平台并发写同一个 session，会话历史被覆盖丢失；按 session key 加串行队列后解决。
- Discord 附件 URL 带签名参数会过期，Agent 事后引用直接 404；现在收到媒体先落本地，再传本地路径给模型。
- 平台重推事件会造成重复回复，用 message id 做幂等键去重。
- Telegram 群聊限速约 1 msg/s，连发三段长文必吃 429；现在合并成单条或主动加间隔。

## 可复用建议

- Adapter 保持“薄”：只做收发与归一，业务逻辑一律放 Agent 层；
- formatter 按通道配置，永远别写进工具里；
- 每条信封打点：平台、端到端耗时、token 用量，账单和瓶颈一目了然；
- 同一个人在两个平台是两个身份，别急着做 identity merge，等真有需求再说。

## 总结

多平台接的是“通道”，不是“大脑”。归一信封、平台级 session key、每通道 formatter——这三件事定下来之后，后面再加一个通道，就是复制一个 adapter 的工作量。先跑通最小链路，再谈打通与优化，顺序别反。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/68d71876d841a9bf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/9288c5b43cc3f848.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/d000f642c0b4d57a.png)

