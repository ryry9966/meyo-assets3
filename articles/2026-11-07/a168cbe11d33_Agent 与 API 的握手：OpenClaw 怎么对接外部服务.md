---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 43714
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

OpenClaw 这类 agent 框架的价值不在于聊天，而在于"动手"：查工单、控设备、同步数据。但 agent 本体只会生成工具调用，真正干活的是背后接入的外部服务。对接质量直接决定 agent 的可用性上限，所以值得把"握手"这件事单独拎出来讲清楚。

## 问题

对接外部服务一般有三条路：MCP server 包 API、插件/skill 提供本地能力、webhook 接收入站事件。选型本身不难，难的是接上之后 agent 的调用是否稳：鉴权怎么管、超时怎么设、错误怎么回传给模型、参数幻觉怎么兜底。常见的通病是"demo 能跑，日常使用三天两头失败"。

## 做法

1. **分层选型**：结构化 API（内部服务、SaaS）走 MCP server，schema 明确；文件、shell、智能家居等本地能力走插件；告警、消息回推等事件流走 webhook，agent 只负责响应。
2. **写好工具定义**：工具名用"动词+宾语"（如 `query_order`、`create_ticket`）；description 是写给模型看的 prompt，要说明"何时用、何时别用"；参数 schema 标注 required 和 enum。
3. **鉴权收口**：密钥放 env 或配置文件，绝不写进工具描述；用户级服务用 OAuth，token 刷新放在 server 侧，不进对话上下文。
4. **超时与重试**：每个工具显式设 timeout；写操作要求幂等键；重试逻辑放在 server 层，别指望模型自己"再试一次"。
5. **结构化错误**：失败时返回 `{error_code, hint}` 而不是原始 HTML 或整段 JSON，模型才能依据 hint 自救。
6. **观测先行**：记录每次调用的工具名、耗时、参数摘要和结果长度，排障时这是唯一线索。

## 踩坑点

- description 太含糊 → 模型乱传参数或选错工具；
- 工具返回大 JSON → 一次调用吃掉大半上下文，先在 server 里裁剪或摘要；
- POST 不幂等 + agent 重试 → 重复下单，写操作必须做幂等；
- 一次注册几十个工具 → 选择准确率明显下降，按领域拆 server，单 server 工具数控制在个位数；
- MCP server 启动失败是静默的 → 工具列表为空，agent 表现为"拒绝执行"，此时先查 server 日志，别急着改 prompt。

## 可复用建议

- **少而粗优于多而细**：一个 `search_orders` 比八个底层 endpoint 工具好用；
- **把对接当 API 设计**：工具接口要版本化，改参数即换版本；
- **上线前做回归**：固定一组高频问题跑 dry-run，看工具选择和参数是否稳定；
- **假设依赖会挂**：给 agent 留一条优雅降级的路径（提示用户稍后重试），而不是报错刷屏。

## 总结

agent 与 API 的握手本质是一份契约：schema 是语法，鉴权是身份，超时和幂等是礼貌，结构化错误是沟通方式。这份契约做扎实，OpenClaw 的 agent 才能从"能调用"走到"可依赖"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/c5e0452f4ad6fc7e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/206871b5d20ada8e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/317c311b5ae33de9.png)

