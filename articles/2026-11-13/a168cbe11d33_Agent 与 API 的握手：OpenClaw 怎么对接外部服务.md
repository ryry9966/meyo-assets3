---
title: Agent 与 API 的握手：OpenClaw 怎么对接外部服务
feedId: 44281
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

OpenClaw 本体是一个跑在你自己机器上的 Agent 运行时：有模型、有会话、有消息通道，但它对外部世界一无所知。模型知识停在训练截止日，也拿不到你内部系统、SaaS 后台、数据库里的实时数据。所以"对接外部服务"本质是给 Agent 装手——让它能调接口、查数据、发消息、触发 CI。这个手怎么装，决定了它是能干活的助手，还是只能聊天的壳。

## 问题

实践里最常见的三类失败：

1. **工具面太宽**：把整个 REST API 包成一个 tool，agent 面对几十个参数不知道填什么。
2. **返回值太重**：20KB 的 JSON 原样塞进上下文，几轮对话后 token 就爆了。
3. **副作用没闸门**：一个"发送通知"工具没有确认机制，agent 循环重试时把你客户群发了一晚上。

## 做法

推荐走 MCP 这条路，比裸写脚本插件更可维护：

**1. 切窄工具面。** 每个工具只做一件事，用动词命名：`search_orders`、`create_ticket`。宁可五个窄工具，不要一个万能工具。

**2. 写 MCP server。** stdio 模式最简单：一个 Python/Node 进程，OpenClaw 拉起它，走标准输入输出通信。入参用 JSON Schema 严格约束，能用 enum 就不用自由字符串：

```json
{
  "name": "search_orders",
  "inputSchema": {
    "type": "object",
    "properties": {
      "status": { "enum": ["paid", "shipped", "refunded"] },
      "limit": { "type": "integer", "maximum": 20 }
    }
  }
}
```

**3. description 当成新人文档写。** 它是模型决定"何时调、怎么调"的唯一依据。写清楚：干什么、什么时候该用、什么时候不该用、失败返回什么。

**4. 裁剪返回值。** 在 server 层做投影，只回模型需要的字段加一条 `summary`，完整数据留给后续按 ID 深查的工具。

**5. 注册并灰度测试。** 先只挂只读工具跑真实问题，观察调用参数是否合理，再逐步放开写操作。

## 踩坑点

- **secrets 进了配置文件**：API key 用环境变量注入，配置里只留引用，别把明文提交进 git。
- **错误返回太"温柔"**：上游 429/500 要在 server 层兜住，返回结构化错误和建议动作，否则 agent 会拿异常文本当数据继续推理。
- **description 与实现漂移**：改了接口不更新文档，agent 按旧契约调用，错得莫名其妙——改工具必须连文档一起改。
- **没留调用日志**：入参、耗时、状态码落盘，排障时这是唯一可信的证据。

## 可复用建议

- 只读先行：新服务先只暴露查询工具，稳定一周再上写操作。
- 写操作加确认闸门：涉及花钱、发消息、删数据的，让 agent 先汇报意图，人工点头再执行。
- 一个上游一个 server：别把十个服务塞进一个 MCP server，权限和故障域都分不开。
- 限制重试：server 层控制次数和退避，别让 agent 的"坚持"放大成事故。

## 总结

Agent 与 API 的握手，本质是三份契约：JSON Schema 是语法契约，description 是语义契约，错误处理是故障契约。把三份契约写清楚、测充分、留好日志，外部服务就从"模型听说的东西"变成"Agent 手里真能用的工具"。OpenClaw 的价值不在替你写胶水，而在给你一个干净的挂载点——契约是你的，握手是标准的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/d8af2fa8bf841a1b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/dbbca3e9cbd18d2f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/6e007d8be84688e5.png)

