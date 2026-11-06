---
title: 跨平台消息路由：让同一个 Agent 同时接住 Telegram 和 Discord
feedId: 43671
source: 综合讨论
publishedAt: 2026-11-07
---

# 背景

Agent 跑起来之后，用户不会只待在一个平台：我们社区里 Telegram 和 Discord 的用户几乎各占一半。与其维护两套 bot 代码，不如让同一个 Agent 核心同时挂两个消息源。下面这套双平台路由在我们环境里稳定跑了几个月，记录一下架构取舍。

# 问题

难点不在"多接一个 API"，而在三件事：

1. **消息模型不同**：Telegram 有 reply_to 和 forum topic，Discord 有 guild/channel/thread 三层结构，各写一套迟早复制粘贴失控。
2. **会话归属**：同一个用户在两个平台找 Agent，上下文要不要共享？
3. **出站约束不同**：长度上限、Markdown 方言、限流策略全都对不上。

# 做法

1. **统一内部消息模型**。定义 schema：`platform / chat_ref / user_ref / msg_id / reply_to / text / attachments`。适配器只做归一化和发送，不写业务。
2. **适配器保持薄**。Telegram 开发期用 long polling，上线切 webhook；Discord 走 gateway WebSocket，纯出站连接，不需要公网入口。
3. **路由与会话键**。`session_key = platform + 范围 + 身份`：群聊按 chat 隔离，私聊按 user 隔离；跨平台共享记忆做成配置项，默认关闭——实测打开后上下文串味很明显。
4. **单队列解耦**。两个适配器往同一个 inbox 投递，Agent 核心只认队列不认平台；回复携带 `reply_via` 元数据，router 据此选 out 站通道。核心挂的 MCP 工具与平台无关，两边能力天然一致。
5. **渲染器按平台分开**。Markdown 方言、长度切分（Telegram 4096 / Discord 2000）、mention 格式各写一个 renderer；切分按段落边界走，避免劈开代码块。
6. **出站限流独立做令牌桶**，失败进重试，最终失败写 dead letter 日志。

# 踩坑点

- Discord 默认不开 MESSAGE CONTENT intent，bot 收到的 text 是空的，排查半天发现是开发者后台没勾选项。
- Telegram MarkdownV2 转义是无底洞，最终放弃完整 Markdown，只保留代码块和加粗子集，其余按纯文本发。
- 用 editMessageText 做流式输出，1 秒改一次就 429；改成 ≥1.5s 间隔，或只在结束时发整段。
- Discord 附件 URL 带签名会过期，归一化时必须立刻下载转存，只存链接等于埋雷。
- Telegram webhook 返回非 200 会重推，Agent 侧要按 msg_id 做幂等去重，否则同一条消息会答两遍。
- 两平台共享记忆时注意并发：同一 session 的请求要串行化，否则上下文交错。

# 可复用建议

- 适配器保持"傻"，逻辑全部下沉到 router 和 renderer。加第三个平台（Slack、飞书）只是再加一对适配器 + 渲染器。
- 日志从第一天就带 `platform:chat:user` 关联键，跨平台排障全靠它。
- 先做 dry-run 模式，把归一化后的内部消息原样打印出来；接新平台时先看这个再接 Agent，能省一半调试时间。

# 总结

这套结构的本质是"平台差异止步于适配层"：Agent 核心从头到尾不知道 Telegram 和 Discord 的存在，路由、渲染、限流彼此独立。跑了几个月，最大的收益不是省代码，而是新增平台时核心一行不动。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/af9d43abd6e681ac.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/390543d76c615dbb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/7e076b94f090bcc1.png)

