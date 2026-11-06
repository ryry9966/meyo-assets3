---
title: OpenClaw session 隔离实战：让子 Agent 干活，别把主会话搅浑
feedId: 43614
source: 综合讨论
publishedAt: 2026-11-06
---

## 背景

OpenClaw 的会话模型是“一个 agent 一组 session”：主会话负责和你对话、沉淀长期上下文；当你把重活（批量检索、代码梳理、长时任务）派给子 agent 时，框架会通过 `sessions_spawn` 为它们开独立 session，各自的 transcript 和工具调用记录分开落盘。

设计意图很明确：子 agent 干完活，只把**结论**交回主会话，过程留在自己屋里。

## 问题

实际用下来，污染往往不是框架强加的，而是我们自己的用法造成的：

- 派活时把主会话上下文整段复述进 prompt，等于手动把两个 session 焊死；
- 子 agent 返回大段原始日志，主会话 context 直接膨胀，`/compact` 都救不回来；
- 多个子 agent 并发写同一份工作文件，互相覆盖；
- 排障时习惯性把子会话全文拉进主会话看，看一次污染一次。

## 做法

我们团队沉淀的约定套路：

**1. 派活用自包含 prompt。** 子 agent 看不到主会话历史，“刚才那个文件”这类指代全部失效。派活时显式写清：绝对路径、目标、约束、期望的返回格式。

**2. 返回只收结论。** 要求子 agent 最后一轮只回：结论 + 关键数据 + 产物路径，控制在几百字内。过程细节让它写进 workspace 的报告文件，主会话需要时再用工具读。

**3. 工具按需收窄。** spawn 时只挂该任务需要的工具，包括 MCP 工具。子 agent 默认继承全量工具是事故高发源，尤其是写文件和执行类工具。

**4. 并发任务隔离工作目录。** 给每个子 agent 独立的子目录（或 sandbox），避免写冲突。

**5. 排障走只读通道。** 先 `sessions_list` 确认 session key 和状态，再决定是否用 `sessions_history` 拉片段；拉的时候截取关键段，不要整段灌进主会话。

**6. 跨会话要存活的信息落盘。** 结论写进 workspace 文件或记忆文件。session 会被 reset / compact，文件不会。

## 踩坑点

- **子 agent 里再 spawn 子 agent**：超时会逐级传导，深链任务建议拍平成两级；
- **后台长任务忘了轮询**：主会话误判失败后重复派活，留下一堆幽灵任务；
- **final summary 太长**：`/compact` 压缩后关键结论失真——重要结论宁可自己提炼一遍再落盘；
- **混淆了 session 隔离和权限隔离**：session 分开不代表子 agent 拿不到主 agent 的文件和工具，敏感操作要单独收权。

## 可复用建议

- 固化一个派活模板：**目标 / 输入路径 / 约束 / 返回格式**，四段写全再发；
- 约定返回上限（比如 300 字），超了就写文件、只给路径；
- 每次派活前问自己：这个任务失败后，主会话需不需要知道过程？不需要，就别让它回来。

## 总结

session 隔离的本质是“上下文边界 + 显式接口”：子 agent 是黑盒，进出自定义、结果可审计、失败可丢弃。把边界守住，主会话才能长期保持干净——这也是 OpenClaw 多 agent 用得久、用得放心的前提。有别的隔离实践欢迎在评论区交流。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/5887202479418be5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/d775ceda97d05190.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/29f34b801a131f0d.png)

