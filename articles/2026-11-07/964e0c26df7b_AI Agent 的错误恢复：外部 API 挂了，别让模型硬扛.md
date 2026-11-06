---
title: AI Agent 的错误恢复：外部 API 挂了，别让模型硬扛
feedId: 43664
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

Agent 的能力边界，基本等于它能调用的工具边界。MCP server、第三方 HTTP API、甚至上游模型本身，都是外部依赖。传统后端聊 API 容错已经聊了很多年，但 Agent 场景有个本质区别：**失败信号最终要交还给一个会"脑补"的语言模型**。这决定了 API 挂掉时，Agent 的故障模式比普通服务更难看。

## 问题在哪

实践里最常见的三种翻车：

1. **模型编造结果**。工具调用超时后返回了空串或原始报错，模型不承认失败，转而"根据上下文推测"补一份看起来合理的数据给用户。
2. **重试失控**。把"要不要重试"交给模型决定，它要么一次就放弃，要么在 429 上无限循环，token 账单先于服务恢复爆掉。
3. **级联拖死**。一个抖动的第三方 API 卡住整个 workflow，后面排队的任务全部超时。

## 怎么做：恢复策略写进代码，而不是写进 prompt

**第一层：工具包装层。** 所有外部调用统一过一层 adapter：设置硬超时（比如 10s，而不是"无上限"）、指数退避加抖动的重试、区分可重试错误（429/5xx/timeout）和不可重试错误（除 429 外的 4xx）。

```python
@retry(retry=retry_if_exception(is_retryable),
       stop=stop_after_attempt(3),
       wait=wait_exponential_jitter(1, 8))
def call_api(...): ...
```

**第二层：错误规范化。** 不要把 stack trace 原样塞进上下文，返回结构化结果：

```json
{"status": "error", "retryable": false, "hint": "upstream auth failed, give up"}
```

模型拿到的是"该放弃还是该换路"的明确信号，而不是一堆它可能复述给用户的噪音。

**第三层：Agent 级预算。** 在 loop 外层卡三本账：最大尝试次数、最大墙钟时间、最大 token 花费。任何一个超了就强制走兜底：读缓存、切备用数据源，或如实告诉用户"这个数据源现在不可用"。

**第四层：熔断。** 同一工具连续失败 N 次后标记为 unhealthy，冷却期内不再发起调用，并提前告知模型"此工具暂不可用，直接走 plan B"。熔断比无脑重试便宜得多。

## 踩坑点

- **200 ≠ 成功**。不少 API 返回 200 但 body 里是 error 字段，规范化层必须校验业务码。
- **幂等性**。对下单、发消息这类 POST 重试前想清楚，加 idempotency key，否则恢复机制本身制造事故。
- **重试不带抖动**。API 一恢复，所有 Agent 同时打过去，第二次雪崩。
- **熔断冷却时间拍脑袋**。太短等于没熔断，太长用户以为 Agent 坏了。建议从 30s 起步，再按失败率调。
- **兜底路径没测过**。降级逻辑平时不跑，真出事时往往自己也有 bug。

## 可复用建议

1. 给每个 MCP 工具配统一的错误契约（status / retryable / hint），Agent prompt 里只描述契约，不描述细节。
2. 永远给模型留一个"声明失败"的合法出口，并在 system prompt 里写明：宁可说不知道，不许编数据。
3. 缓存最近一次成功响应，作为 degraded 模式的数据源。
4. 记录每次工具调用的状态和延迟，失败率是唯一可信的告警信号。

## 总结

错误恢复不是模型的能力，是脚手架的职责。重试、熔断、降级、预算这些决策必须固化在代码里，模型只负责在被约束好的选项中选路。让失败显式、结构化、有边界，Agent 才配得上"自动化"三个字。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/6ff05427b86d90dd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/b8dfdda228c10bef.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/122e49640cc55449.png)

