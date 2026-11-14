---
title: 跨平台消息路由：让一个 Agent 同时值守 Telegram 和 Discord
feedId: 44399
source: 综合讨论
publishedAt: 2026-11-14
---

## 背景

我的 Agent 日常跑在一台小主机上，最初只接了 Telegram，方便随时在手机上喊它查资料、跑脚本。后来协作迁到了 Discord，两个入口来回切很烦，就想让它同时在两个平台在线：谁的消息都能收，也都能答。

OpenClaw 的架构天然适合这种玩法：gateway 是唯一的进程核心，Telegram、Discord 都是挂在 gateway 上的 channel，Agent 本身并不感知“我在哪个平台”。理论上两个 token 一配就完事，实际跑下来还是有不少细节要处理。

## 做法

**1. 配置两个 channel。** 在配置里同时声明 telegram 和 discord，各自填入 bot token，重启 gateway 后双平台同时上线。注意 Discord 侧要在开发者后台打开 Message Content Intent，否则收不到消息正文，只有空事件。

**2. 确认 session 隔离策略。** 默认情况下，每个平台的每个会话是独立的 session key，Telegram 的对话和 Discord 的对话互不串。我建议保持默认——两边的聊天对象、语气、上下文本来就不一样。我一开始想把两边合进同一个 session，结果跨平台上下文一混，回复风格变得很怪，又改了回去。

**3. 让 Agent 知道消息来源。** 在 system prompt 里注明当前消息来自哪个平台，它会自觉遵守该平台的格式习惯：Telegram 少用复杂表格，Discord 可以放开一点 markdown。

**4. 回复路由交给 session 绑定。** session 与来源绑定后，Agent 默认回信到“消息进来的那条路”，这部分框架自己处理得很好，不需要手写路由表。只有主动推送时才需要显式指定目标 channel——我固定让 heartbeat 和系统告警走 Telegram，Discord 只响应消息。

**5. 出站格式做一层裁剪。** Discord 单条消息上限 2000 字符，长回答必须分段；Telegram 的 parse mode 对未闭合实体很敏感，代码块里出现特殊字符可能直接报错。我加了一个出站钩子：超长切片、按平台清理不支持的语法。

## 踩坑点

- **Discord 显示在线却收不到消息**：十有八九是 Message Content Intent 没开。
- **身份是两套**：同一个用户在两个平台是两个不同 ID，allowlist 要分别加，别指望邮箱绑定能自动识别。
- **回声循环**：我曾配置跨平台转发（Telegram 群消息同步到 Discord 频道），结果 Agent 两边都响应，消息来回弹。转发和响应逻辑必须二选一，明确“谁听谁的”。
- **长消息静默截断**：Discord 超限不报错、直接吞，只能靠主动分片。
- **本地调试冲突**：本地起第二个实例时忘了杀掉第一个，Telegram 的 polling 被 getUpdates 冲突顶掉，表现为消息时有时无。

## 可复用建议

- session 默认按平台隔离，除非有强需求再合并。
- 出站消息统一过一道“平台适配钩子”，别让 Agent 直接输出最终格式。
- 两个平台的 allowlist、日志前缀、通知策略分开维护，排查问题时按 channel 前缀过滤日志效率高得多。
- 先单平台跑稳，再加第二个，不要一次全开。

## 总结

一个 Agent 服务多平台，成本主要不在“接通”，而在格式适配和身份权限这两块脏活。保持 session 隔离、守住回复路由的单一出口、把平台差异收敛到一个出站钩子里，双平台值守就能稳定跑。下一步我打算把通知路由也做成配置化，让 heartbeat 能按规则分发到任意 channel。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/2d747474f0a35dbd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/53ff265adc612658.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/22282004920127cb.png)

