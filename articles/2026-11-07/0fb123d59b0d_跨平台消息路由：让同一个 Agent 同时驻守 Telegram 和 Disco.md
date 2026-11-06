---
title: 跨平台消息路由：让同一个 Agent 同时驻守 Telegram 和 Discord
feedId: 43662
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

我有一个跑了几个月的 OpenClaw 实例，原本只接了 Telegram，用来做个人提醒、移动端快速查询和日常脚本编排。但团队协作都在 Discord，开发频道里经常需要把 Agent 拉进来做日志排查、结对读代码。于是问题从"要不要接两个渠道"变成了"接上之后怎么不乱"。

OpenClaw 的架构优势在这里体现得很直接：agent 层和 channel 层是解耦的，加一个渠道不需要动 agent 本身。真正的工作量在路由策略上。

## 问题

实际要解决的是三件事的互相纠缠：

1. **会话隔离**：同一个大脑，两边对话不能串台；
2. **平台差异**：消息长度、Markdown 渲染、回复语义完全不同；
3. **节奏控制**：速率限制、群里什么时候该说话、定时任务发到哪里。

## 做法

**1. 双渠道接入**。在 gateway 配置里同时启用两个渠道，各用各的 bot token（键名以你当前版本文档为准）：

```json5
{
  channels: {
    telegram: { botToken: "<tg-token>", dmPolicy: "allowlist" },
    discord: { token: "<dc-token>" }
  }
}
```

Agent 和 MCP 工具集完全复用，两个渠道共享同一批能力，这是"一个 Agent"的前提。

**2. 会话键设计**。保持默认的按 `渠道:会话ID` 隔离，不要手动把两个渠道指到同一个 session。跨平台共享的应该是 workspace 里的长期记忆文件，而不是对话上下文。

**3. 出站适配层**。写一个小的统一发送函数：按平台限制分片（Telegram 单条 4096 字符，Discord 2000），代码块统一走 fenced block，收窄到两边都支持的 Markdown 公共子集。Agent 产出规范中间格式，适配层负责落地。

**4. 唤醒策略**。Discord 频道用 mention 触发，Telegram 群同理，私聊直答。避免 bot 在群里对每条消息都接话。

**5. 观测**。日志里给每条消息打 channel 标签，两个渠道各留一个只属于自己的调试私聊。

## 踩坑点

- **Discord 2000 字符限制**：长报告直接发送报错。分片函数要作为出站必经路径，不能靠 prompt 里写"请控制在 2000 字以内"——模型不保证守约。
- **Markdown 语法差异**：Telegram 的 parse_mode 和 Discord 渲染规则不同，部分链接语法一边正常一边裸露源码。收敛到公共子集后问题消失。
- **会话串扰**：最初为了"共享记忆"把两边指到同一 session，结果 Telegram 里的私人上下文出现在 Discord 回复里。回退到默认隔离，共享只走记忆文件。
- **速率限制**：Telegram 单聊约 1 msg/s、全局 30 msg/s；Discord 按 bucket 限流。巡检类批量任务必须排队发送，循环硬发会被限流打断。
- **回复语义**：Discord thread 里发起的任务，回复要落回 thread 而不是主频道；Telegram 群的 topic 同理。回复目标必须绑定消息来源。
- **定时任务双发**：定时摘要如果感知"当前渠道"会两边都发一遍，显式指定目标渠道。

## 可复用建议

- 把平台差异当成**出站适配器**问题解决，永远不要写进 system prompt；
- 会话键用 `渠道:chat:user` 三段式，宁可细不可粗；
- 新渠道先跑一周**只读影子模式**（只记日志不回复），确认路由符合预期再放开写权限；
- 让限制最严格的平台（这里是 Discord 的 2000 字符）决定你的分片函数设计，其他平台自然兼容。

## 总结

这套结构的本质是"一个大脑，多个出口"：Agent 保持平台无关，所有脏活——分片、格式、限流、回复目标——都压在适配层。做完之后两个渠道体验一致，维护成本并没有翻倍，因为 90% 的差异被收敛在了一个几百行的发送函数里。如果你的 OpenClaw 实例也有多渠道需求，建议从会话隔离策略开始设计，而不是从接入开始。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/b3057b0d4321ead5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/dbecdce7c35b9095.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/07c5d9d4a1b9e3b4.png)

