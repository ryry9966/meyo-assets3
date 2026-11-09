---
title: MCP 协议入门：把 M×N 的工具接入收敛成 M+N
feedId: 43890
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

写过 OpenClaw 插件的人，大概率经历过这种事：同一个内部查询接口，为了适配不同的 Agent 宿主，得各写一遍工具描述、参数校验和错误处理。各家框架自定义 tool calling 格式，数据源又五花八门，接入成本是 M×N 的——M 个宿主 × N 个工具，就是 M×N 份胶水代码。

## 问题在哪

Function calling 只解决了「模型怎么调工具」，没解决「应用怎么接工具」。Schema 格式、能力发现、生命周期、鉴权，全靠各宿主自己约定。结果就是：工具被锁死在单个应用里，换宿主就要重写；团队里同一个数据库，三个同事写了三套接入。

## MCP 解决了什么

MCP（Model Context Protocol）本质上是一层基于 JSON-RPC 2.0 的薄标准化，2024 年底开源后陆续被主流 Agent 客户端支持。核心模型分三层：

- **Host**：Agent 应用本体，比如 OpenClaw 或桌面客户端；
- **Client**：Host 内维护的连接，一个 Host 可同时挂多个 Server；
- **Server**：能力提供方，暴露三类原语——Tools（可执行动作）、Resources（只读数据）、Prompts（模板）。

M×N 由此收敛成 M+N：工具方实现一次 MCP Server，任何兼容宿主都能发现并调用。发现机制是协议内置的——客户端连上后先 list tools，拿到带 schema 的工具列表再决定怎么用。

## 做法：跑通一个最小 Server

1. **选场景**：优先抽「多个宿主都要用」的能力，比如内部知识库查询、CI 状态查询。只有单一宿主用的，直接写插件反而更省事。
2. **用官方 SDK 起步**：Python 选 FastMCP，几十行能跑。参数用类型注解声明，description 写清楚「什么时候该用这个工具」。
3. **本地开发走 stdio 传输**，远程部署再换 Streamable HTTP。
4. **接宿主前先过 MCP Inspector**（官方调试器）：工具能否被列出、参数校验是否生效、错误返回是否可读。
5. 最后接入 OpenClaw 做端到端验证。

## 踩坑点

- **stdio 模式下别往 stdout 打日志**。stdout 是协议通道，一行 print 就能把整条流搞废。日志一律走 stderr。
- **工具描述就是提示词**。模型靠 description 决定何时调用、怎么传参，写得含糊调用就乱。这部分的投入比代码本身更值。
- **控制返回体积**。一个工具把几万 token 的 JSON 扔回上下文，一次调用烧掉小半个窗口。分页、过滤、截断要在 Server 侧做。
- **错误返回结构化文本**，让模型能读懂并重试；别抛异常崩掉会话，也别甩堆栈。
- **安全别裸奔**。MCP Server 以你的权限运行，第三方 Server 别不加审查就接；工具返回内容也可能携带注入指令，敏感操作保留人工确认。

## 可复用建议

- 一个 Server 对应一个领域，别做万能大杂烩，权限和演进都好办；
- 读数据用 Resources，改状态用 Tools，改状态的工具尽量做成幂等；
- 把 MCP Server 当产品维护：版本号、变更记录、schema 向后兼容；
- 协议问题在 Inspector 里查，别占用 Agent 会话调试。

## 总结

MCP 不会让 Agent 变聪明，它解决的是管道标准化：工具写一次，处处可接。如果只有单一宿主、少量工具，插件就够了；一旦工具需要跨宿主复用，或团队要维护统一的能力层，MCP 的投入就开始回本。建议从一个真实的内部查询场景起步，先跑通 stdio 全链路，再决定要不要往远程部署走。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/be8c357dbda8a5c5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/56975f202979f224.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/fd85309e737ad317.png)

