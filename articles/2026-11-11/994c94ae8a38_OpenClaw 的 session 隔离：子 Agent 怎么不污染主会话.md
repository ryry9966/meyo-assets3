---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 44110
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景

OpenClaw 的主会话（main session）通常绑定你的 IM 渠道，它同时承担两个角色：对话界面，和 Agent 的上下文存储。你在聊天窗口里看到的每一轮记录，都是模型下一轮要重新读一遍的“记忆”。

这意味着主会话里出现的任何东西——工具输出、子任务中间态、插件心跳——都在消耗上下文预算，并直接影响后续对话质量。

## 问题

实际跑起来最常见的三类污染：

1. **工具输出刷屏**：主 agent 直接跑长命令、爬数据、读大文件，几十 KB 的输出进了上下文，之后的对话明显“变笨”。
2. **子任务中间态回流**：派出去的子 agent 把完整过程写回主会话，用户端全是噪音，模型端被细节淹没。
3. **自动化写主会话**：cron 任务、webhook 插件往主 session 塞消息，一旦触发 reset 语义，连带把正常对话也清掉了。

## 做法

核心原则一句话：**物理隔离，摘要上浮。**

1. **按职责拆 agent**。main agent 只做路由、澄清和汇总；重活（检索、长任务、代码生成）交给专职 agent。每个 agent 在 `~/.openclaw/agents/<id>/sessions/` 下有独立的 session 空间，天然互不可见。
2. **子 agent 用隔离运行**。主 agent 通过 spawn 类工具（如 `sessions_spawn`）派生运行，子 agent 拿到自己的 session key，工具调用、报错、中间产物全部落在子 session 里，结束时只把结论回传父会话。
3. **自动化独立绑定**。cron 和 webhook 不要复用 main agent，每个定时任务给一个独立 agent id（比如 `cron-reporter`），跑完后主动把一段摘要发回主渠道，而不是把执行过程写进主会话。
4. **审计与排查**。用 `openclaw sessions list` 看当前 key 分布，`sessions history` 查 JSONL 落盘记录；发现某个 session 膨胀，及时 compact 收缩或直接换新，主对话不受影响。

## 踩坑点

- **靠 prompt 求 agent“少说话”没用**。上下文里该进的还是会进，隔离要靠 session 边界，不靠模型自觉。
- **子 agent 的 channels 配错**，最终回复也会广播到主渠道，出现“同一件事说两遍”的双写。子 agent 建议不绑渠道，只走回传。
- **resetTriggers 设了太口语化的词**。比如把“重开一下”设成触发词，日常聊天意外命中，整个 main session 被清空。触发词必须足够“命令式”。
- **workspace 是隐性的第二上下文**。多个 agent 共享目录、互相读写 NOTES 和状态文件时，污染会以文件形式跨 session 传播。拆 agent 时把 workspace 一起拆开。
- **compact 用晚了**。垃圾已经进了摘要，等于永久污染。膨胀征兆一出现就做，别等上下文爆。

## 可复用建议

- 定死团队约定：**细节下沉**（子 session / 文件），**摘要上浮**（父会话）。大输出先写文件、只回传路径。
- 一个自动化任务一个 agent id，命名带用途前缀，`sessions list` 上一眼可读。
- 每周跑一次 session 审计：看 key 数量、单 session 体量、最后活跃时间，清掉僵尸 session。
- 涉及钱、涉及不可逆操作的活，永远放子 agent 里跑，并要求回传结构化结果，主会话只负责确认。

## 总结

Session 隔离不是一个配置项，而是一种边界纪律：主会话只保留“决策需要的信息”，其他一切留在各自的 session 里。OpenClaw 的 per-agent session 空间和 spawn 隔离运行已经把机制给足了，剩下的是使用习惯——拆得足够细，回传得足够少，你的主会话才能长期保持清醒。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/616d5ed389f4cf72.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/b9d14ad003eb6410.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/6c9017c0617e1329.png)

