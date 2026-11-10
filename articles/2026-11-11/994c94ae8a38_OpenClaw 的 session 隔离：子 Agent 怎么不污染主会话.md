---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 44065
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景

OpenClaw 的 agent 默认跑在一个主 session 里：对话历史、工具调用记录、系统注入全部堆在同一份上下文中。日常单线对话没问题，但一旦让 agent 干“脏活”——批量翻文档、跑长脚本、做多轮试错——中间产物就会直接写进主会话。

## 问题

污染的典型表现：

- 主会话上下文快速膨胀，compaction 频繁触发，早期对话细节被压掉；
- 无关的工具日志稀释模型注意力，后续回答质量肉眼可见地下滑；
- 排障困难：想回看某次子任务的执行过程，只能去主会话流水里捞。

根子一句话：**过程数据和结论数据没有分层。**

## 做法

1. **用 `sessions_spawn` 派生隔离 session。** 子 agent 拿到独立 session key（通常带 subagent 标记和自定义 label）、独立上下文，所有中间过程发生在它自己的会话里，主会话只看到一次任务启动和一次结果回传。
2. **约束回传内容。** 任务描述里写死：只返回结论 + 关键数字；长文本、原始日志落到 workspace 的独立目录（如 `/workspace/tasks/<label>/`），主会话只引用文件路径。
3. **deliver 开关想清楚再开。** 默认让子任务结束只投递一段摘要；全文回传等于白隔离。
4. **给子 agent 限权。** 任务描述里声明禁止动作：不再递归 spawn、不给用户发消息、不动配置类工具。
5. **设置生命周期。** spawn 时指定超时和 cleanup 策略，跑完即收，避免僵尸 session。
6. **上下文显式且最小化注入。** 子 session 看不到主会话历史，把用户偏好、硬约束写进任务说明书，但只给完成任务所需的最小集合。

## 踩坑点

- deliver 开了但没限制输出长度，子 agent 一话痨，污染原样回归；
- 子 agent 与主 agent 共享 workspace，并发写同一个文件互相覆盖——约定子任务一律写独立子目录；
- 图省事把主会话历史整段贴给子 agent，隔离意义归零；
- 忘了 cleanup，session 列表越积越长，事后排查成本陡增；
- 递归 spawn 没有深度上限，任务树失控。经验值：一到两层足够，再深基本是设计有问题。

## 可复用建议

核心原则就一句：**子 agent 负责过程，主会话只保留结论。**

- 任务描述当成“工作说明书”来写，四段齐全：目标、边界、输出格式、禁止动作；
- 大产物一律落盘，会话内只传路径和摘要；
- label 命名带任务语义（例如 `research-docs-0508`），事后用 `sessions_history` 定位不抓瞎；
- 定期用 CLI 的 sessions 列表巡检一遍，顺手清理僵尸。

## 总结

session 隔离不是什么高级特性，本质是把“过程”和“结论”分开存放：派生时少给一点上下文，回收时少传一点内容，主会话就能长期保持干净、可预测。下次让 agent 干脏活之前，先试着把任务说明书写清楚——写不清楚的任务，隔离了也照样会漏。

（注：`sessions_spawn` 的具体参数名以你当前版本的工具 schema 为准，行为逻辑各版本基本一致。）

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/713bf1a04e234c63.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/8f2a2f5e8e2eb8cf.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/90c18d845f941a87.png)

