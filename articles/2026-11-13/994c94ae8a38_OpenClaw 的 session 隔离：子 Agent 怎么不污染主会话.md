---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 44300
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

OpenClaw 的主会话是一条长跑的上下文循环：用户消息、模型回复、工具调用结果全部堆在同一个 session 里。跑得越久，context 越满——网页抓取的原文、排障日志的中间输出、成片的文件读取结果，都会永久占据 token 预算。表现出来就是：响应变迟钝、模型开始"忘记"早先的指令、费用曲线越来越陡。

OpenClaw 给出的解法是 session 隔离：重活下放给 sub-agent，在独立 session 里跑，主会话只收结论。

## 问题

主会话被污染的典型场景：

- 让主 agent 亲自读三份长文档做对比，读入内容全部进主上下文；
- 排查一个 bug 连环调用工具，几十条中间输出把关键指令挤到窗口边缘；
- 群聊场景下任务上下文和闲聊混在一起，无法收拾。

这些不是"模型不行"，而是我们把**执行现场**和**决策现场**混在了同一个 session。

## 做法

1. **派生而不是亲自干**。主 agent 通过 `sessions_spawn` 起 sub-agent，把任务描述、必要上下文、期望输出格式写进 task 参数。
2. **上下文要显式传递**。sub-agent 是全新 session，看不到主会话历史。相关结论、文件路径、约束条件必须在 task 里写全，别指望它"自己知道"。
3. **要求紧凑返回**。在 task 里明确"最终只返回结论和关键证据，不要贴中间过程"。sub-agent 的返回值会作为一条消息进入主会话，这是唯一的回流通道。
4. **收窄子会话的能力面**。配置里给 sub-agent 限制工具白名单（比如只读不写）、指定更便宜的模型、设置执行时限。
5. **需要回查时显式去取**。用 `sessions_history` / `sessions_send` 跨会话读取或传递，默认不共享。

示意：

```json
{
  "tool": "sessions_spawn",
  "params": {
    "task": "对比 docs/a.md 与 docs/b.md 的差异，只返回变更点清单与风险标注",
    "model": "便宜档模型",
    "executionTimeLimit": "300s"
  }
}
```

## 踩坑点

- **假设 sub-agent 有记忆**。它不知道主会话聊过什么，task 里少写一句话，它就多跑十次工具。
- **返回值太肥**。不约束输出格式时，sub-agent 会把整段摘录塞回来，隔离等于白做。
- **workspace 是共享的**。会话上下文隔离，但工作区文件、持久记忆共用。两个 sub-agent 同时写同一文件会互相覆盖，按任务拆目录。
- **递归派生**。默认不允许 sub-agent 再 spawn；如果放开过这个限制，警惕深度失控。
- **把"拉历史"当习惯**。频繁用 `sessions_history` 把子会话全文搬回主会话，污染只是换了个入口。

## 可复用建议

- 判断标准很简单：**会产生大量中间输出的任务，就该下放**。阅读型任务（调研、对比、扫日志）和试错型任务（改代码跑测试）是典型。
- 把高频任务固化成 task 模板：写清输入、边界、返回格式，主 agent 只做填空。
- 定期用 `/new` 重置主会话，配合隔离机制，别让主会话变成永久垃圾场。

## 总结

Session 隔离的本质是把"执行"和"决策"分开：主会话保持薄、保持稳定，只承载对话和判断；sub-agent 是一次性的执行现场，用完即弃，只回传结论。**上下文不共享、workspace 共享、回流只走一条窄通道**——记住这三件事，sub-agent 就不会变成新的污染源。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/0bbb85de87988373.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/e664c9348cf9dcf5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/56078497723a0062.png)

