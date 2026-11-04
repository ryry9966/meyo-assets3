---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 43458
source: 综合讨论
publishedAt: 2026-11-04
---

# 背景

在 MCP 出现之前，想把一个外部能力（数据库、文件系统、某个 SaaS API）接进 LLM 应用，通行做法是：在每个应用里手写一层 function calling 胶水代码。你有 3 个 Agent 项目、10 个想接的工具，就是 30 份相似但不相同的适配代码，schema 格式、错误处理、鉴权方式全靠各自发挥。

MCP（Model Context Protocol）做的事情很朴素：把"模型应用如何发现和调用外部工具"标准化成一个开放协议（Anthropic 于 2024 年底开源），基于 JSON-RPC 2.0，定义了 Host（Agent 客户端）和 Server（能力提供方）之间的消息格式与生命周期。类比的话，它更像 LSP 之于编辑器——编辑器不用为每种语言写集成，语言服务端实现一次协议即可。

# 它具体解决了什么

1. **M×N 集成收敛为 M+N**：工具方写一次 server，应用方实现一次 client。
2. **工具描述格式统一**：每个工具带结构化 JSON Schema 和面向模型的描述，host 聚合后注入上下文，供模型做函数选择。
3. **三类原语分工清晰**：Tools（可执行动作）、Resources（只读数据）、Prompts（预置模板），不再混在一张工具表里。
4. **传输与生命周期标准化**：本地走 stdio，远程走 Streamable HTTP，initialize 阶段做能力与版本协商。

# 上手步骤

以最小可用为例：

1. 选官方 SDK（Python 或 TypeScript），用 stdio 传输起一个本地 server。
2. 定义 2~3 个工具，参数用严格 JSON Schema，description 写给模型看。
3. 用 MCP Inspector 连上自测：工具列表、参数校验、返回结构逐项过一遍。
4. 在宿主（OpenClaw、Claude Desktop 或自研 client）里注册 server，确认工具已注入上下文。
5. 在 agent 循环里把模型发出的 tool call 路由回对应 server，处理结果并回填。

# 踩坑点

- **stdout 污染**：stdio 传输下 stdout 就是协议通道，任何 `print` 或日志打到 stdout 都会破坏消息流。日志一律走 stderr。
- **描述写得含糊，模型就选错工具**：description 本质是 prompt 的一部分，按"模型会怎么读"来写，不是给人看的注释。
- **Schema 太松**：大量 optional、枚举不收口，会显著抬高参数幻觉率。能 required 就 required。
- **长耗时工具**：同步阻塞会拖死整个循环，注意超时与进度反馈。
- **工具数量失控**：十几个 server、上百个工具全量注入，先爆上下文窗口，其次崩的是选择准确率。要做聚合与过滤。
- **安全别抱侥幸**：装一个第三方 MCP server，等于授权 Agent 执行一段任意代码。文件系统 server 记得收窄 root、优先只读。

# 可复用建议

- 按领域拆 server（github、db、fs 各一个），别写万能大杂烩。
- 工具粒度对齐"用户会说出的一个动词"：太细碎模型要自己编排，太粗则没有选择空间。
- 只读数据优先用 Resources 暴露，动作才用 Tools。
- 把 Inspector 自测固化成调试习惯：出问题先分清是协议层故障还是模型选错了工具。

# 总结

MCP 不会让模型变聪明，它把"接工具"从一个个项目里的定制工程，变成一次性的标准实现。真正的收益在复用：server 写一次，跨项目、跨宿主都能用。如果你正在维护多个 Agent 项目的工具胶水层，现在是把它收敛成 MCP server 的合理时机。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/8f4643b345b33c95.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/6e47aa72d6604708.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/67ea86c43ebbd56f.png)

