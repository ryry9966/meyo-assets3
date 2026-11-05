---
title: OpenClaw session 隔离实战：让子 Agent 干脏活，主会话保干净
feedId: 43526
source: 综合讨论
publishedAt: 2026-11-05
---

# 背景

用 OpenClaw 跑日常自动化的应该都有体会：主会话是最容易"发胖"的部分。每次工具调用的原始输出、多轮来回、失败重试，全堆在同一个 session 里。跑得越久，上下文越臃肿，模型越容易跑偏，token 账单也越难看。

而很多任务本质上是"一次性脏活"——批量读 20 个文件、抓一组页面、跑一轮测试。你只关心最终结论，不关心过程。如果这些中间产物全部写进主会话，主会话就被污染了。OpenClaw 的解法是子 Agent + session 隔离。

# 问题：什么算"污染"

主会话里出现以下任何一种，都算被污染：

- 子任务的工具原始输出（几 KB 的日志、HTML 片段）
- 子任务失败重试的中间痕迹
- 子 Agent 自己的推理过程

轻则挤掉主会话里真正重要的上下文，重则让主 Agent 后续决策被无关细节带偏。

# 做法：隔离的正确姿势

核心机制一句话：**子 Agent 有独立的 session 文件，主会话只收结论。**

1. 用 `sessions_spawn` 派生子 Agent。每个子 Agent 在 `agents/<agentId>/sessions/` 下有自己独立的 transcript，**不继承主会话历史**。
2. 派发时把"任务所需的一切"写进 task prompt：目标、约束、可用工具、返回格式。子 Agent 是白纸开局，别指望它读心。
3. 结果回传走 `sessions_send`，只把最终摘要写回主会话。在 task 里显式约定："只返回结构化结论，不超过 300 字，不贴原始输出。"
4. 干完即清理：用 `sessions_delete` 删掉子 session，或至少把 transcript 归档。
5. 验证：`sessions_list` 看活跃 session，`sessions_history` 抽查子 Agent 实际干了什么——排查问题时这两条必须熟。

需要更强隔离时，给子 Agent 单独配 sandbox/workspace 和工具白名单，避免它顺手用了只有主 Agent 该有的权限。

# 踩坑点

- **以为子 Agent 能看到主会话上下文。** 看不到。主会话里的约定、偏好、前情，都得在 task prompt 里重新交代，否则子 Agent 会按"通用默认"干活。
- **回传不设上限。** 子 Agent 一激动把整个 JSON dump 发回来，主会话照样爆炸。返回契约要写死。
- **session 文件只增不删。** 自动化跑一个月，sessions 目录里几十个僵尸 session，排查时全是噪音。建议每次 spawn 带可辨识的 taskID。
- **递归 spawn 不设限。** 子 Agent 再派子 Agent，成本和失控风险指数级上涨。层级最好压在一层。
- **权限继承的错觉。** 沙箱和授权是按 agent 走的，子 Agent 不会自动继承主 Agent 的授权，网络、文件写入都要单独批。

# 可复用建议

- 把主会话当**控制面**，子 Agent 当**数据面**：控制面只留决策和结论，数据面的脏活全部下沉。
- 固化一个"返回契约"模板（结论 / 置信度 / 异常清单三段式），所有 spawn 复用，别每次现编。
- 定期 compaction 或 `/new` 重置主会话，配合归档，别让"长期记忆"变成长期噪音。
- 重要的子 Agent transcript 先归档再删，事后审计能救命。

# 总结

session 隔离不是什么高深特性，本质就是一条纪律：**过程留在子 session，结论才进主会话。** 把这条纪律固化成 spawn 模板和清理习惯，主会话就能长期保持瘦而清醒，自动化跑得越久越稳。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/181858941dda193b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/c5bc14a506894032.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/850530d63a7cedcf.png)

