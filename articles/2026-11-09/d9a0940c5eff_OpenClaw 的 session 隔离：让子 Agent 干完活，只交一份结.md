---
title: OpenClaw 的 session 隔离：让子 Agent 干完活，只交一份结论
feedId: 43859
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

用 OpenClaw 做自动化跑得久一点，几乎都会走到同一步：把耗时的活儿交给子 Agent——批量整理文件、调 MCP 工具链、抓数据、跑长任务。但不少人第一次用完就发现主会话"变笨了"：翻历史全是子任务的工具输出，真正重要的对话被挤没了。有人用 `/new` 重置会话救急，结果把有用的上下文也一起丢了。正确的思路是：让这些任务根本不发生在主会话里。

## 问题：污染的三种典型表现

- **上下文膨胀**：几十次工具调用、失败重试的输出堆在主会话，有效信息密度骤降。
- **状态串台**：子 Agent 顺手改了共享的 memory 文件，把中间结论当"事实"写进主 Agent 的记忆，之后的对话全被带偏。
- **文件互踩**：子 Agent 和主 Agent 读写同一个 workspace，产物、草稿、日志混在一起。

## 做法：把过程关在子 session 里

OpenClaw 的默认设计就是"过程与结论分离"，要点五条（具体配置以你版本的文档为准）：

1. **用 spawn 拉起子 Agent**，不要在主会话里手动跑长任务。子 Agent 拿到独立 session，上下文从零开始，只带你在 spawn prompt 里写明的任务描述。
2. **run-to-completion**：子 Agent 自己跑到结束，不中途打扰主会话。
3. **只回传一份 result**：默认只有最后一条总结消息注入主会话，过程性 tool call 留在子 session，主模型看不到。
4. **工作区分开**：给子 Agent 配独立 workspace 目录，产物落在它自己的目录里。
5. **高风险任务加沙箱**，限制文件系统与网络范围。

spawn prompt 建议固定成四段：目标、输入位置、输出位置、返回格式。信息给到"够完成任务"为止。

## 踩坑点

- **spawn prompt 塞太多父上下文**。隔离的意义直接归零，还可能把主会话里的隐私带进子 session。
- **子 Agent 与主 Agent 共用 memory 路径**。它跑完把中间结论写进记忆，主会话下次开场就被带偏。要么指到子 Agent 自己的目录，要么在任务描述里明确禁止写 memory。
- **result 写太长**。"只回传结论"不等于能回传三千字——注入主会话的就是这条消息。约定"不超过 5 句结论 + 产物路径"。
- **假设子 Agent 继承主 Agent 的全部工具权限**。权限按 agent 配置走，派活前先确认。
- **定时任务直接写主 session**。cron 类任务同样会污染，让它们也走子 session。

## 可复用建议

- 给所有子 Agent 定统一的返回模板：结论、产物路径、异常与未完成项，三项缺一不可。
- 产物一律落盘，主会话只持有文件路径，需要时再让主 Agent 去读，而不是把内容塞回对话。
- session key 按任务和日期命名，事后用 `/sessions` 或日志排查能对得上号。
- 子 session 的历史留着是为了排查问题，定期清理，别让旧任务残留影响判断。

## 总结

session 隔离的本质是**过程与结论分离**：OpenClaw 用独立 session 加只回传 result 完成了机制层面的事，但真正不污染主会话，取决于你管住三件事——spawn prompt 的信息量、子 Agent 的写入路径、返回消息的长度。把这三条固化成模板写进你的 agents 配置，子 Agent 才能安心当"干活的"，主会话始终负责"做决定的"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/dd1e88066ee8ebea.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/e83b7e63c6aaf01f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/0f514bc754923ef9.png)

