---
title: OpenClaw 的 session 隔离：让子 Agent 干脏活，主会话保持干净
feedId: 43655
source: 综合讨论
publishedAt: 2026-11-06
---

OpenClaw 的主会话是按 session key 持久化的 JSONL 记录（默认在 `~/.openclaw/agents/<agentId>/sessions/` 下），它同时承担三个角色：对话上下文、compact 之后的长期摘要来源、排障时的第一现场。也就是说，主会话里留下的每一行，都会在之后的对话中持续计息。

## 问题：污染是怎么发生的

最常见的场景：你对主 agent 说"帮我整理一下下载目录"。它当场在主会话里连跑几十条 shell，每条输出都写进 session。后果依次出现：

- 上下文迅速膨胀，请求变贵、变慢；
- 提前触发 compaction，早期对话被摘要稀释，关键约定丢失；
- 调试信息残留，后面聊天时模型容易被一堆旧报错带偏；
- 日志里的密钥、内网路径长期躺在主会话文件里。

主会话应当是"指挥官"，不该亲自干脏活。

## 做法

1. **先做隔离判定。** 满足"一次性、过程输出大、失败可重试、结论能一段话交付"的任务进子 agent；需要和用户反复确认的交互式流程，留在主会话。

2. **用 `sessions_spawn` 派发。** 注意子 agent 看不到主会话历史，task 必须自包含：背景、工作目录、约束、交付物一次写全。可以先用主会话把上下文压缩成一段，再塞进 task。

3. **约定返回协议。** spawn 的返回值会原样进入主会话，且超长会被截断。所以规定子 agent：结论先写文件，最终回复只给"结论 + 变更路径 + 异常"，控制在两三百字。重要细节去文件里看，不靠回传。

4. **权限收敛。** 给脏活配独立的 agent 定义：独立 workspace、只读工具白名单。就算子 agent 跑偏，可破坏面也小。

5. **审计与回收。** `sessions_list` 看有哪些会话，`sessions_history` 抽查、或在子任务失败时读取现场；用完的子会话定期清理，别让会话列表本身变成新的噪音源。

## 踩坑点

- "继续刚才那个事"式的 task 是无效的，子 agent 会一脸懵——历史不在它那边。
- 让子 agent 反问用户：主会话里看不到它的提问上下文，交互直接错乱。
- 子 agent 之间用 `sessions_send` 互相同步，会形成网状依赖，排障时找不到状态在哪。保持星型结构，只向主会话汇报。
- 子 agent 里再 spawn 子 agent，一层以内可控，两层以上回传链路和耗时都不可控。
- 只 spawn 不清理，一个月后 sessions 目录里躺着几十个孤儿会话。

## 可复用建议

- 四段式 task 模板：**背景 / 目标 / 边界（能做什么不能做什么）/ 交付格式**。
- "结论落盘、回复给路径"：把回传当成摘要接口，而不是数据通道。
- 把隔离规则写进 workspace 的 `AGENTS.md`：哪些任务必须 spawn、返回格式是什么，让主 agent 自己遵守，而不是每次靠人提醒。
- 配合 `/compact` 和 `/new` 控制主会话体量，清理节奏和子会话回收放在同一个维护动作里做。

## 总结

OpenClaw 的 session 隔离不是花哨功能，而是一个预算问题：上下文就是预算，过程噪音花得越多，留给真正决策的注意力就越少。让子 agent 在自己的 session 里折腾，只把摘要交回来，主会话保持轻——这是让长期运行的 agent 不越用越笨的关键。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/21067e49efbe77b5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/0aaaa6b3e4783e9f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/bc46b9c171b99333.png)

