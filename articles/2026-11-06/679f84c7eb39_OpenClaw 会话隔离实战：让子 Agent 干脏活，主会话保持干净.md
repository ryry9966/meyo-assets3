---
title: OpenClaw 会话隔离实战：让子 Agent 干脏活，主会话保持干净
feedId: 43607
source: 综合讨论
publishedAt: 2026-11-06
---

## 背景

OpenClaw 的主会话是和你长期对话的那条上下文：系统提示、AGENTS.md、记忆文件、历史消息全在里面。跑自动化任务时，最容易犯的错误是让主会话亲自去干重活——翻几十个文件、调一串 MCP 工具、批量执行命令。几次下来，上下文就被一次性任务的细节塞满了。

## 问题：什么叫"污染"

- 工具输出和中间推理留在主会话，触发 compact 之后，真正重要的指令被摘要稀释；
- 一次性任务细节永久占据历史，token 成本持续上升；
- 子任务的状态、临时文件、甚至加载过的 MCP 工具 schema 残留在主会话；
- 角色漂移：模型本该做调度，却开始"顺手"自己执行，行为不可预测。

## 做法：主会话调度，子会话执行

核心思路一句话：**主会话只做拆解、派发和验收，执行全部丢进隔离子会话。**

1. **用 `sessions_spawn` 派生子会话。** 子 agent 拿到全新上下文，只带上明确的任务描述，跑完后仅回传最终结果。中间的 exec 输出、read 的文件内容、MCP 调用记录都留在子会话自己的 jsonl 里。
2. **写"返回契约"。** spawn 时在 task 里明确要求：只回结论 + 产物路径，控制在几百字以内。不要让它把完整日志贴回来。
3. **工作区隔离。** 子任务要写文件时，在 spawn 参数里指定独立 workspace 或子目录，避免多个子 agent 互相覆盖。需要更强隔离就上 sandbox（Docker），exec 和文件系统都不碰宿主。
4. **会话路由。** 多渠道/多项目场景下，在 `openclaw.json` 里把不同 agent 绑定到不同 session key，物理分开，别让所有对话挤一条主线。
5. **定期体检。** `sessions_list` 看各会话体积，主会话异常增长就 `/new` 或 `/compact`，子会话跑完即弃。

## 踩坑点

- **回传结果太大。** spawn 解决了过程污染，但子 agent 返回一份 3000 字报告，主会话照样膨胀。产物落盘，只回路径，字数卡死。
- **串话。** 用 `sessions_send` 时 session key 写错，子 agent 的回复直接进了主聊天渠道，用户视角一片混乱。发送前先 `sessions_list` 确认目标。
- **启动上下文继承过多。** 子 agent 会继承 AGENTS.md 等基础内容，如果这份文档很长，每个子会话的固定成本都不低。长文档按需 read，别全量堆在启动上下文里。
- **并行写冲突。** 多个子 agent 同时改同一个仓库，git 状态互相踩。要么串行派发，要么各自 checkout 到独立目录。

## 可复用建议

- 把"是否需要 spawn"当默认判断：预期超过 3–4 次工具调用的任务，都值得开子会话；
- 主会话里只保留三样东西：任务拆解、派发记录、验收结论；
- 为高频子任务沉淀固定的 task 模板（含返回契约），减少每次手写；
- 定期看 `~/.openclaw/agents/*/sessions/` 下 jsonl 的体积，它就是污染程度的直接指标。

## 总结

Session 隔离不是什么高级技巧，本质是把上下文当作需要预算管理的资源。主会话是常驻大脑，子会话是用完即弃的手套。OpenClaw 已经把 spawn、send、workspace、sandbox 这些隔离件备齐了，剩下的只是养成习惯：派发前想清楚回传什么，跑完顺手清理。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/1b6afcb6909e1fdf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/bcfd6ce96611911d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/f607aadd0dd796d8.png)

