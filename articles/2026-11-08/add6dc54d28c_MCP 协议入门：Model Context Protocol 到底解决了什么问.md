---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 43758
source: 综合讨论
publishedAt: 2026-11-08
---

## 背景：从“到处写适配器”说起

做过 Agent 或自动化流程的人大概都有类似经历：模型本身不难接，难接的是模型周围的“世界”——文件系统、数据库、浏览器、内部 API、各种 SaaS。每接一个新数据源，就要写一套鉴权、错误处理、schema 定义和重试逻辑。这类胶水代码写多了，会发现它们结构高度相似，只是接口不同。

MCP（Model Context Protocol）就是针对这个现状提出的开放协议：用一套标准化的消息格式（基于 JSON-RPC 2.0），规定 AI 应用（Host）与外部能力提供方（Server）之间怎么发现、描述和调用彼此。

## 问题：M×N 的集成泥潭

没有统一协议时，集成成本是 **M×N**：M 个 AI 应用 × N 个工具，理论上要写 M×N 个连接器。MCP 把它压缩成 **M+N**——Server 侧按协议实现一次，Host 侧按协议实现一次，中间即可互通。

协议核心只有三个原语，分清它们是入门第一步：

- **Tools**：模型决定何时调用，通常有副作用（写库、发请求）
- **Resources**：宿主决定注入上下文的数据，只读
- **Prompts**：用户主动触发的模板

大多数误用，源于把这三类混为一谈。

## 做法：一个最小可用的 Server

以 Python 官方 SDK 为例，暴露一个查询工具只需十几行：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo")

@mcp.tool()
def get_order(order_id: str) -> str:
    """按订单号查询订单状态，返回状态与更新时间。仅支持 2024 年后的订单。"""
    return query_db(order_id)

mcp.run()  # 默认 stdio 传输
```

实操路径：

1. **先想清楚能力分类**：只读数据走 Resource，有副作用的走 Tool，别混。
2. **写最小 Server**，先只暴露一个工具，跑通再扩展。
3. **描述写给模型看**：参数含义、返回结构、适用与不适用场景。这直接决定调用准确率。
4. **本地调试用 MCP Inspector**，能可视化看到握手、列出工具、手动调用。
5. **接入宿主**（Claude Desktop 或自研 Agent），本地用 stdio，远程用 Streamable HTTP。

## 踩坑点

- **工具描述含糊 = 调用灾难**。模型要么不调，要么乱传参数。把描述当成给新同事写的接口文档来写。
- **工具数量膨胀**。几十个工具塞进上下文，模型选择错误率明显上升。按场景分组或动态加载。
- **stdio 的环境问题**。Server 进程继承宿主环境，路径、Python 版本、环境变量差异是常见故障源，Windows 上尤其多。
- **远程部署的状态陷阱**。Streamable HTTP 是有状态会话，直接按无状态 serverless 部署会踩 session 的坑，状态请放外部存储。
- **安全别省**。工具返回内容会拼进模型上下文，天然是 prompt injection 入口。写操作加确认，权限最小化。

## 可复用建议

- 把工具接口当产品做：一个工具只做一件事，返回结构稳定、可截断。
- 大返回值做分页或摘要，别一次打爆上下文窗口。
- 给每次工具调用记入参和耗时——排查 Agent“为什么这么干”时，这是唯一可靠证据。
- 用 Inspector 做回归：改了描述先手动调一遍再上线。

## 总结

MCP 解决的不是“让模型更聪明”，而是**接入的标准化**：把 M×N 的集成成本压成 M+N。对 OpenClaw 插件生态来说，这也是目前把能力暴露给各类 Agent 的性价比最高的方式。真正难的从来不是跑通协议，而是设计出好用的工具界面和克制的上下文管理——这部分没有协议能替你做。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/8a27135a0f6a9bd9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/b3b30f9a2a4aca1e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/1dda29b042418062.png)

