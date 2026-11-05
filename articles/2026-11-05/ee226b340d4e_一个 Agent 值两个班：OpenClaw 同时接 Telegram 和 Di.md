---
title: 一个 Agent 值两个班：OpenClaw 同时接 Telegram 和 Discord 的路由实践
feedId: 43540
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

OpenClaw 的结构是“网关 + 多渠道适配器 + 单一 Agent 内核”。单机部署时，大多数人只挂一个渠道。但我们的实际情况是：一半同事泡在 Telegram，另一半在 Discord，谁也不肯搬家。与其两边各跑一个 Agent 实例（意味着两套记忆、两份工具配置、双倍维护），不如让同一个 Agent 进程同时服务两个平台，共享同一套 MCP 工具和长期记忆。

## 问题

“连上两个 SDK”不是难点，真正要解决的是三件事：

1. **会话归属**：同一个人在两边发消息，Agent 应该当作一个人还是两个人？上下文要不要合并？
2. **格式差异**：Telegram 的 MarkdownV2 转义规则和 Discord 的 markdown 完全不同，长度上限也不同（4096 vs 2000）。
3. **事件噪声**：Discord 服务器里有大量与 Agent 无关的闲聊，不能全喂给模型烧 token。

## 做法

**第一步：双渠道声明。** 在网关配置里同时声明 telegram 和 discord 两个 channel，各填 bot token。Discord 侧务必在开发者后台打开 Message Content Intent，否则收到的消息内容恒为空——这是最常见的“假连接”，网关显示在线但什么也读不到。

**第二步：会话路由策略。** OpenClaw 默认按 channel + chat_id 隔离 session，两边各聊各的，这对多数场景够用。如果确实需要跨平台连续上下文，加一层 identity map：维护 `(platform, user_id) → canonical_user` 的映射表，把同一个人的消息路由到同一个 session。我们用 SQLite 存这张表，包成一个工具挂给 Agent。

**第三步：平台感知提示词。** 入站时向 system prompt 注入当前来源平台，让 Agent 知道“这条来自 Discord 群”，回复时自觉控制长度、少用表格。

**第四步：出站两段式渲染。** 不要让模型直接产出各平台的最终文本。统一内部中间格式，在网关出站层按目标平台渲染：Telegram 侧处理转义，Discord 侧做长度检查和分段。

## 踩坑点

- **Discord Intent 没开**：症状是“收到消息事件但 content 为空”，排查半天以为是 token 权限问题。
- **Telegram chat_id 类型**：群组 ID 以 `-100` 开头，私聊是正数。做 identity map 时平台前缀别漏，否则会发生 key 冲突。
- **共享 session 的并发写**：如果强行让两个渠道写同一个 session 文件，会互相覆盖。解法是串行化队列，或者退一步只按“人”隔离、不按“平台”合并。
- **提及归一化**：Discord 的 `<@id>` 格式直接透传给模型会污染上下文，在入站层先替换成用户名。
- **静默截断**：长回复在 Discord 超限不报错、只截断，出站层必须主动检查长度。

## 可复用建议

- **渠道适配层做薄**：业务逻辑全部收进 Agent 工具和 MCP server。我们后来接第三个平台（Slack）只写了一个适配器，花了一下午——复利就在这里。
- **别把平台语法写进 prompt**：让模型负责 MarkdownV2 转义永远不靠谱，格式问题一律工程层解决。
- **identity map 留手动绑定接口**：自动按用户名合并很容易认错人，宁可多一步确认。
- **每个 channel 独立记日志和心跳**：单边掉线时能第一时间发现，而不是等用户来报障。

## 总结

跨平台消息路由的核心不是“接两个 SDK”，而是把**身份、会话、格式**三件事从 prompt 里挪到工程层。OpenClaw 的网关结构天然适合这种分工：渠道只管收发和渲染，Agent 内核保持唯一，技能与记忆只此一份。两个平台跑稳之后你会发现，多平台不是功能问题，而是边界划分问题——边界划对了，扩展就是体力活。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/d3793b2ddfdb1746.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/a8b49bc434263262.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/d212acf356c82b16.png)

