---
title: AI Agent 的错误恢复：当外部 API 挂了怎么办
feedId: 44083
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景

Agent 和传统服务最大的区别，是调用链里多了一个不确定的"决策者"。外部 API 一挂，不只是请求失败——LLM 可能拿着错误信息做出各种不可控动作：无限重试、编造数据、把报错原文复读进上下文。在 OpenClaw 的典型场景里，MCP server 和插件封装的外部服务（搜索、数据库、第三方 SaaS）恰恰是最常见的故障源。

## 问题的三种形态

1. **工具层失败**：超时、429、5xx、鉴权过期。
2. **语义层失败**：HTTP 200 但业务报错，或返回空结果——Agent 会把空结果当真。
3. **上下文污染**：重试几轮后，context 里堆满报错原文，模型被带偏，开始复述错误甚至放弃任务。

## 做法

核心原则：**重试策略放在代码层，不要交给模型决定**。LLM 对"该重试几次"没有稳定判断，每次决策还消耗 token。

1. **错误分类**。在工具封装层把错误分成三类：可重试（超时、429、502/503）、不可重试（401/403、400 参数错误）、语义失败（200 但结果异常）。
2. **有界重试 + 指数退避 + 抖动**。比如最多 3 次，1s/2s/4s 加随机抖动，写死在 wrapper 里。
3. **熔断**。连续 N 次失败后快速失败，冷却期内不再打上游，避免雪崩和长阻塞。
4. **结构化错误返回给模型**。不要把堆栈原文丢进 context，返回紧凑 JSON：

```json
{"status": "error", "code": "UPSTREAM_TIMEOUT",
 "retryable": false, "hint": "服务暂不可用，可改用缓存数据或告知用户"}
```

`retryable` 和 `hint` 两个字段能显著减少模型乱来。

5. **降级阶梯**。重试失败后按序尝试：备用工具 → 过期缓存（标注数据时间）→ 明确告知用户失败原因。宁可诚实失败，不要让模型脑补一个结果。
6. **上下文卫生**。多轮重试后做一次压缩摘要，把报错细节收敛成一句"该工具当前不可用，已降级"。

wrapper 的思路大概十行：

```python
def call_tool(fn, *args, max_retries=3):
    for i in range(max_retries):
        try:
            r = fn(*args)
            return r if r else semantic_error(r)
        except RetryableError:
            sleep(backoff(i) + jitter())
    return structured_error("UPSTREAM_TIMEOUT")
```

## 踩坑点

- **非幂等操作不要盲目重试**。发消息、下单类工具重试会重复执行，要么上游支持幂等键，要么重试前先查状态。
- **用 per-call 超时，而不是全局超时**。一个挂死的 MCP 工具卡住整个 agent loop 是高频事故。
- **429 别当 5xx 处理**。限流要读 Retry-After 退避更久，否则越重试越挂。
- **空结果 ≠ 无数据**。搜索返回空列表时，模型经常编一个"看起来合理"的答案，空结果也要走结构化错误路径。
- **重试日志带 request id**，否则事后排障和上游对不上。

## 可复用建议

- 给每个 MCP 工具统一套一层 wrapper：分类、重试、熔断、结构化返回，一处实现全局受益。
- 维护一份"降级阶梯"清单：每个关键工具有没有备用源或缓存？没有就先补上。
- 做故障注入回归：手动把某个工具改成必超时，验证 agent 是优雅降级而不是无限重试。
- 监控重试率和熔断触发次数——这是上游健康度最直接的信号。

## 总结

错误恢复的本质，是把不确定性关进代码的笼子：分类、有界重试、熔断、结构化上抛、明确降级。模型只负责在给定选项里决策，不负责发明恢复策略。做到这一点，外部 API 挂掉时，你的 agent 表现会从"抽风"变成"体面"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/250c63f1d75e2ea9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/6758beb187ee7531.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/3556bbaa397a2c5b.png)

