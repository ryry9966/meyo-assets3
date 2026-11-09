---
title: 一个 Agent 同时服务 Telegram 和 Discord：双通道路由实践
feedId: 43912
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

我这边的主 Agent 一直只挂在 Telegram 上，服务自己和一个小群。后来几个协作的伙伴只用 Discord，就面临选择：再起一个独立 Agent 实例，还是让同一个 Agent 同时接两个平台。我选了后者，跑了两周多，把过程和坑整理出来。

## 问题

多开一个实例看似省事，实际代价不小：

- 两份配置、两套会话，同一个问题要问两遍；
- 长期记忆分裂，一边记住的事另一边不知道；
- 工具和 MCP 配置要维护两份，改一处漏一处。

所以目标很明确：**单实例、单份记忆、双通道接入**。

## 做法

**1. 网关层挂双通道，Agent 层保持无感知。** 在 channels 配置里同时启用 telegram 和 discord，各自填 bot token。关键是不要在系统提示词里写死任何平台特征，让网关负责消化所有平台差异。

**2. 会话按「平台 + 会话 ID」隔离。** Telegram 按 chat_id，Discord 群聊按 channel_id、私信按 user_id 生成 session key。不要默认把两个平台合并进同一个会话——上下文串台比记忆分裂更难收拾。

**3. 输出统一用中性 Markdown，适配器负责渲染。** Agent 只产出标准 Markdown，Telegram 侧转 HTML parse mode，Discord 侧基本原生支持。超长回复由网关按 4096 / 2000 字符上限切分。

**4. 触发规则分开配。** Discord 群里用 @mention 触发，避免刷屏；Telegram 私信直接响应，群聊用回复或关键词。这些全部写在网关的路由配置里，不进代码。

**5. 工具和记忆挂在 Agent 层。** MCP server、文件记忆、定时任务都是单份，两个平台共享——这是这次改造真正的收益所在。

## 踩坑点

- **Discord mention 要清洗。** 消息体里会带 `<@id>` 形式的原始标记，直接喂给 Agent 会污染上下文，网关层必须先剥掉。
- **Telegram MarkdownV2 转义很痛。** 我最后放弃 MarkdownV2，改用 HTML parse mode，代码块和链接基本无损。
- **切分长回复别切断代码块。** 简单按长度切会把 ``` 围栏劈开，先按代码块边界分段，再用长度兜底。
- **并发要串行。** 两个平台同时来消息时，Agent 可能被并发调用。按 session 排队，或给单个 Agent 加运行锁，否则工具调用会出现竞争。
- **语音消息两边格式不同。** Telegram 是 ogg/opus，Discord 走附件，转写管线要兼容两种来源。

## 可复用建议

- 适配器只做「翻译」，不做业务逻辑；Agent 保持平台无关。
- 路由规则（谁能触发、在哪触发）全部配置化，改起来不用动代码。
- 日志带上 channel 标签，排障时一眼区分来源。
- 上线前用两个小号在私聊里各测一轮，再放进群。

## 总结

这次改造代码量不大，大部分工作在网关配置和适配器细节上。核心结论：**Agent 保持无感知，把平台差异全部下沉到网关层**，就能用一份记忆和一套工具同时服务两个平台。后续如果再接 Slack 或飞书，理论上只需要多写一个 channel 适配器。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/ff179bd375ae6f67.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/330513b0572e3fad.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/6981c515a0763024.png)

