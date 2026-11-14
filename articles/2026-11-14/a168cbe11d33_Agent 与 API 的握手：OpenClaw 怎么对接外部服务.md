---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 44389
source: 综合讨论
publishedAt: 2026-11-14
---

## 背景

Agent 的价值最终落在“能动手”上：查工单、拉数据、发通知。在 OpenClaw 里，这条链路通常分两层——底层是 Connector（纯 HTTP 封装），上层是 Tool / MCP Server（面向模型的接口）。很多团队一开始直接把原始 API 文档塞给模型，结果调用频繁失败、上下文爆掉、副作用失控。这篇文章记录一次对接内部 ERP 接口的实践，供同类场景参考。

## 问题

对接外部服务时，反复出现三类问题：

1. **认证与配置散落**：API key 写死在配置或 prompt 里，既不安全也无法多环境切换；
2. **接口语义不清**：模型不知道什么时候该调哪个工具、参数怎么填，选错工具率明显偏高；
3. **失败处理缺失**：超时、限流、非幂等操作重试，轻则 Agent 卡死，重则产生重复订单。

## 做法：四步握手

**第一步：先写 Connector，再写 Tool。** Connector 是无状态 HTTP 客户端，base URL、超时、重试、认证注入都收在这一层。工具层只做两件事：把模型的自然语言参数翻译成结构化请求，把 API 返回裁剪成模型能读懂的摘要。

```yaml
tools:
  - name: search_orders
    description: 按订单号或时间范围查询订单，返回最多 20 条摘要；失败时返回 ok=false
    connector: erp_api
    timeout_ms: 8000
    params:
      order_id:  { type: string, required: false }
      date_from: { type: string, required: false }
```

**第二步：认证走环境变量。** 插件配置里只放引用（如 `${ERP_API_KEY}`），实际值由部署环境注入。配置文件可以进仓库，密钥不进仓库、也不进模型上下文。

**第三步：把 description 写成“可执行级别”。** 不写“查询订单”，而是写清楚适用场景、参数约束、失败时的返回形态。模型选工具几乎完全依赖这段描述，值得花时间打磨。

**第四步：统一错误出口。** Connector 把 4xx / 5xx / 超时归一化成 `{ok, code, message, hint}`。`message` 给模型看，`hint` 明确告诉它下一步该重试、换参数还是放弃，避免模型在失败后自由发挥。

## 踩坑点

1. **返回体太大撑爆上下文**：原始 JSON 一把梭，几次调用就把窗口吃满。Connector 里必须做字段裁剪和分页截断。
2. **对非幂等接口盲目重试**：重试只给 GET 和幂等写操作；下单、发消息要么带幂等键，要么干脆不重试。
3. **工具描述太抽象**：“处理数据”这类描述会让模型在相近工具之间乱选。
4. **密钥进了对话上下文**：调试时把 header 直接打进日志，等于把 key 送给模型记忆，日志也要做脱敏。
5. **没有 dry-run**：上线前没在沙箱环境跑一轮真实调用，第一个线上故障往往来自权限没开通。

## 可复用建议

- 一个工具只做一件事，参数面尽量小；复杂流程让 Agent 多次调用组合，不要堆万能接口。
- 分层之后，上游接口变更只需要改 Connector，工具描述和 Agent 侧不用动。
- 每次工具调用记一条结构化日志：工具名、参数摘要、耗时、结果码，排障时按 trace id 串起来。
- 给核心工具写契约测试：mock 上游返回，断言裁剪后的输出结构稳定，防止上游悄悄改字段。

## 总结

对接外部服务不是“接个 API”这么简单，而是给 Agent 设计一套清晰的感官和肌肉：Connector 管可靠传输，Tool 管语义清晰，配置管安全边界，三层各司其职，握手才算握稳。下次你发现 Agent 调工具总是失败，先别急着改 prompt——从工具描述、错误出口和超时配置这三处查起，通常十分钟能定位到根因。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/8ad9a3d66ff895fb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/7c114ce65e5dae69.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/7700ddde69e497ae.png)

