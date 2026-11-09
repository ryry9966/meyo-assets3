---
title: 跨平台消息路由：一个 Agent 同时服务 Telegram 和 Discord
feedId: 43966
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景

我们的运维助手 Agent 最初只挂在 Telegram 上，几个同事在群里用得还不错。后来协作迁到了 Discord，问题来了：再起一个 Agent 实例，意味着两份配置、两份记忆、两边答案不一致。更合理的结构是——**一个 Agent 运行时，多个渠道适配器**。OpenClaw 的 gateway 架构天然支持这个思路，这次把 Telegram 和 Discord 同时接上，把过程记录下来。

## 问题

真正要解决的不是“能不能连上”，而是三件事：

1. **入站归一化**：两个平台的消息结构、提及语法、附件引用完全不同；
2. **会话与记忆边界**：同一个 Agent，哪些上下文共享、哪些按渠道隔离；
3. **出站渲染**：Agent 输出标准 Markdown，但 Telegram 的 MarkdownV2 转义和 Discord 的方言、长度限制（4096 / 2000）都不一样，直接透传必炸。

## 做法

**第一步：统一消息信封。** 在渠道插件层把入站消息压成统一结构：`{channel, chat_id, user_id, text, reply_to, media[]}`。Agent 内核只认识这个信封，不感知平台差异。

**第二步：会话键设计。** 我们用 `channel:chat_id` 作为会话键，群聊和私聊天然隔离；长期记忆放在 Agent 共享工作区，两个平台读同一份知识。跨平台身份映射（同一个人在两边）单独存一张小表，按需合并，不要默认合并。

**第三步：路由规则。** Telegram 群里只在 @提及 或回复 bot 消息时触发；Discord 走 mention + 斜杠命令双入口。触发后的处理逻辑完全一致，只是入口不同。

**第四步：出站渲染分离。** Agent 输出保持纯 Markdown，由每个渠道自己的 renderer 转换：Telegram 用 HTML parse mode（原因见踩坑），按 4096 切片；Discord 按 2000 切片，并保护代码块完整性。

```yaml
channels:
  telegram:
    adapter: polling      # 本地开发无公网，先用轮询
  discord:
    adapter: gateway      # websocket，不需要 webhook
agent:
  session_key: "channel:chat_id"
  shared_memory: true
```

## 踩坑点

- **Telegram MarkdownV2 转义是深渊**：下划线、点、括号全要转义，LLM 输出几乎必然漏。换 HTML parse mode，只处理 `<` `>` `&`，稳定得多。
- **Discord 切片别按字符硬切**：会切断代码块。先按代码块边界分段，再按长度切。
- **Message Content Intent 忘开**：Discord 后台没开这个 intent，bot 只能收到空消息，排查了半小时才发现。
- **媒体不互通**：Telegram 的 file_id 和 Discord 的 CDN URL 不能互传，走 Agent 的文件工具重新下载上传。
- **会话串台**：最初只按 user_id 做会话键，同一个人两个平台上下文混在一起，换成 `channel:chat_id` 才干净。

## 可复用建议

- 新增渠道只写 adapter + renderer，不碰 Agent 内核，信封结构保持稳定；
- 上线前用一个 CLI 假渠道做 dry-run，先验证路由和渲染逻辑，再接真 bot；
- 出站投递失败按 correlation id 记日志，跨平台排障时能对上号；
- bot token 全部走环境变量，别进仓库。

## 总结

跨平台的核心不是“多接一个 bot”，而是把差异挡在适配器层：**入站归一化、会话显式设计、出站分渠道渲染**。Agent 内核保持无平台感知之后，下次要加 Slack 或企业 IM 渠道，成本就只剩写一个新的 adapter。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/10bbbee1c24da885.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/287f255a1b0dc9d3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/4bb3b36b17d0c674.png)

