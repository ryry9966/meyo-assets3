---
title: 一个 Agent 同时服务 Telegram 和 Discord：跨平台消息路由实践
feedId: 43552
source: 综合讨论
publishedAt: 2026-11-05
---

# 一个 Agent 同时服务 Telegram 和 Discord：跨平台消息路由实践

## 背景

我们使用群里约六成用户在 Telegram，四成在 Discord。最初的做法是两个入口各跑一个实例，很快失控：提示词改版只同步了 TG，Discord 那边的工具列表也漂了。OpenClaw 的 Gateway-channel 模型本身支持多 channel 接入，于是我们花了一个周末把它收敛成「一个 Agent 核心 + 两个 channel 适配」。这篇记录关键决策和踩坑。

## 问题

真正要解决的只有三件事：

1. **会话策略**：同一个人在两个平台是两个身份（ID 体系完全不同），上下文共享还是隔离？
2. **消息形态**：Discord 有 embed、2000 字符上限、线程和 forum；Telegram 是 MarkdownV2 和严格的转义规则。
3. **限流差异**：两边 429 的语义、退避方式不同，长回复切片策略也不同。

## 做法

- **单一核心，薄适配**。Gateway 同时挂 telegram 和 discord 两个 channel，各自 token；channel 层只负责收发、鉴权、格式渲染，业务逻辑不下沉。
- **统一内部信封**。Agent 与 MCP 工具只看到规范化消息结构（发送者、来源 channel、会话键、内容、附件引用），不感知平台。
- **会话键默认隔离**：`{channel}:{userId}`。跨平台记忆做成独立插件：显式绑定两平台身份，留审计日志，默认关闭。隐私上宁可麻烦一点。
- **渲染层分离**。Agent 输出统一 Markdown 内部格式；Discord 侧按段落切块（上限留 buffer）、优先 embed；Telegram 侧严格转义后走 MarkdownV2，失败降级纯文本。
- **出站统一队列**。按 channel 配置速率，读限流响应头做退避，用来源 message id 做幂等去重。

示意配置（字段名以实际版本 schema 为准）：

```yaml
channels:
  telegram: { botToken: "..." }
  discord:  { botToken: "..." }
routing:
  sessionKey: "{channel}:{userId}"
renderers:
  discord:  { splitAt: 1900, preferEmbed: true }
  telegram: { parseMode: MarkdownV2, escape: strict }
```

## 踩坑点

- **Telegram 转义**：MarkdownV2 连代码块内部都要转义，最常见死法是模型输出带下划线的文件名。先严格转义，出错整条降级纯文本重发。
- **Discord 切片**：硬切会把代码块劈成两半、渲染崩坏，务必按段落和代码块边界切。
- **上下文串台**：早期试过默认共享记忆，结果 TG 私聊内容出现在 Discord 群回复里——这就是改成默认隔离的直接原因。
- **线程来源**：forum 帖的事件带 threadId，别当普通频道消息处理，否则回复会飞到主频道。
- **长任务反馈**：超过 10 秒的任务先发一条中间状态，Discord 用户对「没反应」容忍度极低。
- **幂等**：网络重试造成过同一问题回答两遍，去重键必须落在出站层，不是 agent 层。

## 可复用建议

- 判断逻辑归属：与**平台**有关的进 renderer，与**人**有关的进 agent，两边都不该出现对方的代码。
- 内部信封 + renderer 的结构，之后接 Slack 或 Matrix 基本只需新写一个 renderer。
- 先只开一个平台跑两周，日志统一打 channel 标签，再开第二个。
- 身份绑定/映射类操作必须可审计、可撤销。

## 总结

跨平台路由的复杂度不在收发消息——那部分 OpenClaw 已封装得很好——而在会话策略和格式渲染。经验浓缩成三句话：适配层保持薄，平台差异关在 renderer 里；会话默认隔离，共享必须显式；出站队列统一管限流与幂等。照这个结构走，接第三个平台是加法，不是重写。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/dbe29c93fcecb063.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/fad59f220de0c9fd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/bf78c0eff8bb2910.png)

