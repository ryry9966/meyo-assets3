---
title: OpenClaw 的 session 隔离：让子 Agent 干活，主会话只留结论
feedId: 44271
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

用 OpenClaw 跑了一段时间后，最明显的问题不是模型不够聪明，而是主会话越来越"重"。定时任务抓的网页、MCP 工具的调试输出、顺手让主 agent 做的一次性整理，全部写进同一个 session 文件。上下文一满，要么触发压缩丢细节，要么手动 /new 重来。session 隔离解决的就是这个：把"干活"和"记事"分开。

## 问题：主会话被当成垃圾桶

OpenClaw 的每段对话对应一份 session 转录，模型每轮看到的上下文就是它。凡是写进主 session 的内容——工具原始输出、失败重试日志、子任务的中间步骤——之后每一轮都会被重新送进模型。代价是三层的：token 成本线性上涨；指令遵循变差，长上下文里模型容易抓错重点；重置会话时，真正重要的状态（长期约定、偏好）也被一起清掉。

## 做法：主会话只做控制面

我的原则一句话：主会话是控制面，只保留决策和结论；重活全部丢给子 Agent 的独立 session。

三步配置（openclaw.json）：

```json5
agents: {
  defaults: {
    sessionScope: "per-sender",   // 会话按发送者隔离
    subagents: { mode: "auto" }   // 允许模型按需派生子 Agent
  }
}
```

**第一步，确认 sessionScope。** per-sender 让不同联系人的上下文互不可见；群聊场景可换 per-channel，代价是话题容易混，按场景取舍。

**第二步，派生而不是亲历。** 让主 agent 对重活说"spawn 一个子 agent 去做"，OpenClaw 会通过 sessions_spawn 起一个全新 session：子 agent 看不到主会话历史，它的工具调用和中间过程只写在自己的转录里，父会话只收到它的最终消息。

**第三步，把"返回什么"写进任务描述。** 派生提示词必须约束输出，例如：

> 任务：读取 notes/ 目录，汇总未归档条目。不要展开原始内容，只返回 10 行以内的清单和一句结论。

## 踩坑点

- **把大段上下文塞进派生提示词**，等于换个地方继续污染。正确做法是传文件路径，让子 agent 自己去读。
- **子 agent 的最终消息会原样进父会话**。不约束长度，一份热情的总结照样能把主 session 写满。
- **子 agent 继承同一 workspace 和工具权限**。session 隔离是上下文隔离，不是权限隔离，别当安全边界用。
- **子 agent 默认不知道主会话聊过什么**。需要共享的状态，要么显式写进任务提示，要么落盘成文件。
- **排查方法**：看 ~/.openclaw/agents/ 下主 session 的 jsonl 增长速度。涨得异常快，说明有东西在往里写工具输出，先查定时任务和 MCP 调用是不是跑错了会话。

## 可复用建议

- 给每个自动化任务写固定模板：目标 / 输入路径 / 约束 / 输出格式与长度上限。
- 定时任务的提示词按"只返回结论"来写。每次运行本来就是独立 session，别浪费这个天然隔离。
- 主会话定期 /compact；长期约定沉淀到 workspace 的 MEMORY.md，而不是指望某个 session 永远活着。
- 委派链保持一层。子 agent 再派子 agent，信息会在层层摘要里失真，出问题也难定位。

## 总结

session 隔离本质是上下文卫生：父会话收结论，子 session 吞噪声。配置上只需要 sessionScope 加 subagents 两行，真正起作用的是纪律——派生之前想清楚"我只要什么回来"。跑通之后你会发现，主会话轻了，模型反而更听话了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/3249a6b6f0e597bd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/b62038470ca4026d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/1a8cbc12620534d8.png)

