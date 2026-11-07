---
title: OpenClaw session 隔离实践：别让子 Agent 污染主会话
feedId: 43753
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

用 OpenClaw 跑日常自动化时，主会话（比如绑在 Telegram 上的那个 default session）是长期存活的：系统提示、历史消息、工具结果都在同一个上下文里滚动。为了干活，我们经常让主 Agent 派子 Agent 去跑重活——批量抓页面、跑长脚本、调一串 MCP 工具。

## 问题的样子

如果子 Agent 和主会话共用一个 session，它所有的中间输出——几十条工具调用、报错重试、大段网页原文——都会写进主 transcript。表现就是：上下文迅速膨胀、compaction 频繁触发、主 Agent 开始“忘事”，甚至把子 Agent 的输出风格带进日常回复。我上周就踩过一次：派子任务整理 40 个页面，做完后主会话回复明显变啰嗦，排查半天，发现子 Agent 的浏览器结果全落在了主 session 里。

## 做法

1. **派生时显式隔离**。用 `sessions_spawn` 派子 Agent 时指定 isolated session，而不是让它继承主会话 ID。子 Agent 拿到干净上下文 + 任务描述，干完活只回传一份结果摘要。参数名各版本略有差异，spawn 前用 `/help` 或文档确认。
2. **约定结果契约**。任务描述里写死返回格式：如“只返回 300 字以内要点 + 结论，不贴原始数据”。隔离解决“过程”污染，契约解决“结果”污染。
3. **工具按需挂载**。重的 MCP server（浏览器、爬虫类）只在子 Agent 的 run 里启用，主会话只留轻工具——工具 schema 本身也占上下文。
4. **子会话独立 workspace**。并行多个子任务时各自一个工作目录，避免文件读写互相踩。
5. **记录 session ID**。子会话 ID 落到本地日志，出问题时能单独打开那个 transcript 排查，或直接清理跑飞的子会话。

## 踩坑点

- 最常见的是**忘了 isolated**：默认继承父 session，污染是静默发生的，事后翻 transcript 才发现。
- **摘要不等于没污染**。让子 Agent“把结果写回来”，结果贴了 2000 字，主会话照样膨胀，契约里必须有长度上限。
- **隔离 ≠ 失忆**。子 Agent 看不到主会话历史，别指望它“知道之前聊过什么”，需要的背景要显式写进任务描述。
- 跑飞的子 Agent 会持续烧 token，子任务要设超时，超时后主动清理。
- 共享 workspace + 并行执行 = 偶发文件覆盖，日志和中间产物务必分目录。

## 可复用建议

- 把“隔离子 Agent + 结果契约 + 独立 workspace”做成默认模板，新任务直接套。
- 长期主会话定期手动 compaction 或重开 session，别等它自己挤爆。
- 一句话原则：**只晋升结果，不晋升过程**。过程留在子会话里，需要审计时再翻。

## 总结

Session 隔离不是高深配置，本质是把“干活”和“对话”拆成两个上下文，再用一份受控的摘要连接它们。做之后主会话的上下文增长曲线明显放缓，token 账单和回复质量都会稳很多。如果你也在用 OpenClaw 跑多任务自动化，建议今天就检查一下：你的子 Agent 是不是正在悄悄写主会话。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/92b6b700126ba6b5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/832fdd84317fd36f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/c98b74bfa5d1803b.png)

