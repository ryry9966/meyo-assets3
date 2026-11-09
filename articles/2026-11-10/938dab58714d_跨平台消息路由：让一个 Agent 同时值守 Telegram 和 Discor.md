---
title: 跨平台消息路由：让一个 Agent 同时值守 Telegram 和 Discord
feedId: 43946
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景

我一直把 Agent 当私人助理用，主入口是 Telegram。但团队协作在 Discord，很多临时问题直接在频道里抛出来。两套机器人、两份 prompt、两份记忆，很快就会"精神分裂"：同一个问题，Telegram 上答过一次，Discord 上又答一遍，上下文还不一致。于是花了一个周末，把两个平台收敛到同一个 OpenClaw 实例上。

## 问题

跨平台不是"多接一个 channel"那么简单，核心麻烦有三个：

1. **会话隔离**：同一个人在两个平台是两个身份，默认拆成两个 session，记忆不互通。
2. **消息形态差异**：Discord 有线程和斜杠命令，Telegram 有话题和回复键盘，Markdown 方言也不同。
3. **出站限制不同**：Discord 单条 2000 字符，Telegram 4096；图片、语音的处理路径完全不一样。

## 做法

**第一步：通道接入。** 在 openclaw.json 的 channels 段同时挂 telegram 和 discord，token 走环境变量，不入库。Telegram 用长轮询起步，Discord 走网关连接，记得在开发者后台打开 Message Content Intent，否则收不到群消息正文。

**第二步：身份与会话路由。** 这是整件事的关键。我没有强行合并账号，而是维护一张 identity map：Discord 用户 ID 和 Telegram user ID 映射到同一个业务身份，session 主键用业务身份而非平台 ID。这样上下文共享，但入口仍可区分——prompt 里带上来源平台，Agent 知道自己"在哪说话"。

**第三步：出站渲染分层。** Agent 只输出统一的 Markdown，发送前由各通道渲染器转换：Telegram 走 HTML 转义（MarkdownV2 的转义规则太折磨），Discord 保留原生 Markdown 但按 2000 字符切片。超长回复统一落盘成文件、发摘要加附件，这是两边都适用的兜底。

**第四步：媒体归一。** 入站图片、语音先转成统一临时文件路径，再塞进消息信封交给 Agent，插件不直接碰平台 SDK。

## 踩坑点

- **Telegram 群组隐私模式**：默认收不到普通消息，要么关掉，要么全靠命令触发，早点定。
- **Discord Intent 没开**：机器人"上线了但装聋"，我排查了半小时才反应过来。
- **切片边界**：按字符硬切会把代码块切烂，改成先按段落、再按块级元素切才没出事故。
- **双平台回声**：调试期两个通道都开了群回复，同一句话 Agent 答了两遍。用来源标记 + 群内仅提及触发压住了。
- **限速**：Discord 频道级限速很敏感，广播类消息排队发，别并发硬怼。

## 可复用建议

1. 通道层保持"薄"：只做收发、格式转换和限速，业务判断全部交给 Agent。
2. 消息信封永远带 `platform / chat_id / user_ref` 三个字段，路由和审计都靠它。
3. 身份合并宁可晚做也别做错，identity map 留人工确认入口。
4. DM 白名单/配对机制开起来，别让 bot 裸奔；配置进版本库，token 全走环境变量。

## 总结

跨平台路由的难点不在"接上"，而在会话语义的统一和出站格式的收敛。把身份映射、渲染分层、媒体归一这三件事做扎实，一个 Agent 服务多个平台就退化为纯粹的运维问题。这套结构目前稳定跑了几个星期，预估新增一个通道（比如 Slack）只需要写一个薄适配器。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/699b894e2df13b61.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/900cfd624f0ee83a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/a0eca2dc51aba5c0.png)

