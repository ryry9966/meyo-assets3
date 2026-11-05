---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 43538
source: 综合讨论
publishedAt: 2026-11-05
---

# MCP 协议入门：它到底解决了什么问题

## 背景

做过 Agent 自动化的人大概都有同感：最耗时的往往不是提示词调优，而是“接工具”。OpenClaw 这类本地网关型 Agent 的能力上限，基本取决于它能调到哪些服务和数据——读文件、查库、控制浏览器、发消息。问题是，每样东西的接法都不一样。

## 问题：M×N

在 MCP 出现之前，这基本是一个 M×N 问题：M 个 Agent 宿主 × N 个工具/数据源，每一个组合都要写一份适配——function calling 的 schema 格式不同、插件 API 不同、鉴权方式不同。你给框架 A 写的插件，搬到框架 B 得重写一遍。社区里大量重复造的连接器，就是这么来的。

## MCP 的解法：M+N

MCP 把这个结构拧成了 M+N。它定义了三件事：

- **角色**：Host（Agent 宿主）、Client（宿主内的连接器，与 Server 一对一）、Server（暴露能力的独立进程）；
- **三类原语**：tools（模型决定何时调用）、resources（宿主控制的数据读取）、prompts（用户触发的模板）；
- **传输**：本地 stdio、远程 streamable HTTP。

类比很直白：给 AI 应用统一了 USB-C。工具方实现一次 Server，所有支持 MCP 的宿主都能发现并调用它；宿主方实现一次 Client，就接入了整个生态。

## 做法步骤

以 Python 官方 SDK 为例，一个最小 Server 只需要几行：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("weather")

@mcp.tool()
def get_city_weather(city: str) -> str:
    """查询指定城市的当前天气。city 为中文城市名，失败时返回原因。"""
    ...

mcp.run(transport="stdio")
```

然后在宿主配置里注册这条 stdio 命令，重启后通过 `list_tools` 确认工具被识别，对话里直接让它查天气即可。核心心智模型：**函数签名 + docstring 就是给模型看的 schema**，docstring 不是注释，是接口文档。

## 踩坑点

1. **stdio Server 别往 stdout 打日志**，会污染 JSON-RPC 通道导致握手失败；日志一律走 stderr 或文件。
2. **description 含糊，模型就选错工具或干脆不选**。把参数约束、单位、失败场景写清楚。
3. **工具返回一大坨原始数据会撑爆上下文**，做裁剪、分页或摘要。
4. **远程 Server 别裸奔**，streamable HTTP 场景配好 OAuth/令牌，并保留用户确认环节。
5. **工具挂太多会掉准确率**，还常驻占用上下文，按需启用、分组管理。

## 可复用建议

- 先把内部系统能力包成**领域单一的小 Server**，不要做一个上帝 Server。
- 工具粒度偏向**高层意图**（如 `draft_weekly_report`）而非原子 CRUD，减少模型拼装出错。
- 错误信息返回**可读、可行动**的文本，模型才能自我修正重试。
- Server 保持无状态、可独立测试，用 Inspector 或脚本直接调用，不要只在宿主里盲调。

## 总结

MCP 没有魔法，它做的事是把“Agent 接外部能力”这个接口标准化，把 M×N 降成 M+N，让工具可迁移、宿主可复用。但它不保证效果——工具设计的质量仍然决定 Agent 的上限。建议从一个小而完整的 stdio Server 跑通全链路，再考虑远程化和鉴权，比一开始就上大而全的方案省心得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/5f97961f09c38d2d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/5c42e62b27c35063.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/dafedb75aabcf13e.png)

