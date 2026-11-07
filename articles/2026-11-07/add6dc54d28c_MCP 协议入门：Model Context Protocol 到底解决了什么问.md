---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 43751
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

做 Agent 开发绕不开一个现实：模型本身只会生成文本，真正干活靠的是外部能力——查数据库、调内部 API、读文件、发消息。在 MCP 出现之前，这些能力的接入基本是各写各的：OpenAI 用 function calling 的 schema，LangChain 有自己的 Tool 抽象，Claude 又是一套。每写一个工具，就要针对每个运行时适配一遍，N 个模型 × M 个系统，就是 N×M 份胶水代码。

MCP（Model Context Protocol）是 Anthropic 在 2024 年底开源的协议，目标是把"模型怎么连接工具和数据"这件事标准化。可以类比 USB-C：设备形态千差万别，但接口统一了。

## 它到底解决了什么

严格说，MCP 没有让模型变聪明，它解决的是工程问题：

1. **工具复用**。一个 MCP Server 写好后，任何支持 MCP 的客户端（Claude Desktop、各类 Agent 框架、IDE 插件）都能直接挂载，不用为每个运行时重写。
2. **关注点分离**。鉴权、连接管理、错误处理收敛在 Server 里，Agent 侧只负责决策调用。
3. **动态发现**。客户端连接后能列出 Server 提供的 tools / resources / prompts，工具清单可以运行时变化，不用改 Agent 代码。

## 上手步骤

最小可用路径四步：

1. **选传输方式**。本地进程用 stdio，远程服务用 Streamable HTTP。先从 stdio 开始最省事。
2. **写一个最小 Server**。Python SDK 几行就能跑：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo")

@mcp.tool()
def get_order(order_id: str) -> str:
    """按订单号查询订单状态，返回状态和最近更新时间。"""
    return db.query_order(order_id)

mcp.run()
```

3. **在客户端注册**。配置里声明 command / args / env，客户端启动时拉起该进程。
4. **用 MCP Inspector 调试**，确认工具列表、入参 schema、返回内容符合预期，再接入真实 Agent。

## 踩坑点

实际用下来几个高频问题：

- **工具描述太敷衍**。模型选工具完全依赖 name + description，"处理数据"这种描述会导致选错工具、传错参数。描述要按"给新同事看的 API 文档"标准来写。
- **工具数量失控**。挂 40 个工具后上下文被撑爆，选择准确率明显下降。按场景拆 Server、按需启用。
- **stdio 环境问题**。Windows 下 npx 直跑经常报错，需要包一层 `cmd /c`；Node/Python 版本不一致会导致静默失败，先看 stderr 日志。
- **安全别忽视**。MCP Server 跑在你的用户权限下，接入第三方 Server 前要审代码；工具返回内容若被注入指令，模型可能照做，敏感操作务必加人工确认。

## 可复用建议

- 新集成先做只读工具，跑稳了再放开写操作。
- 一个 Server 只管一个领域（订单归订单，日志归日志），别做成大杂烩。
- 返回结构尽量精简，只给模型需要的字段，省 token 也降错误率。
- 团队内部系统可以收敛到一个内部 MCP Server 统一暴露，比每人各写一遍脚本划算。

## 总结

MCP 的价值不在炫技，而在于把重复的胶水工作变成一次性的标准件。如果 Agent 只接一两个工具，直接写 function calling 也够用；一旦要接的系统多起来，或者希望工具能跨客户端复用，MCP 就是当下最值得投入的那层协议。建议从 stdio + 一个只读工具跑通全链路，再逐步扩展，这是最平滑的路径。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/fd0b8b655a32827f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/94537c27442d4048.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/a49c0fb4306be11a.png)

