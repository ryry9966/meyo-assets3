---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 44069
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景

做 Agent / 自动化的人大概率都写过这样的代码：为了让模型查数据库、调内部 API、操作本地文件，给每个框架手写一遍 function calling 的 schema、鉴权和错误处理。换个模型供应商要改格式，换个 Agent 框架要重写胶水层。工具越多、环境越多，重复劳动按乘法增长——这是典型的 **N×M 集成问题**。

## MCP 到底解决什么

MCP（Model Context Protocol）没有提升模型能力，它解决的是**集成的工程问题**：把"模型如何发现和调用外部能力"抽成一层标准协议。

三个核心角色：

- **Host**：Agent 运行环境（比如你的客户端/编排层）；
- **Client**：Host 内部维护与服务端连接的组件；
- **Server**：暴露 `tools` / `resources` / `prompts` 的独立进程。

协议基于 JSON-RPC 2.0，传输层常用 `stdio`（本地子进程）和 Streamable HTTP（远程部署）。握手完成后，客户端可以列出工具、调用工具、读取资源。收益很直接：**N 个 Host × M 个工具，变成 N + M**。工具方实现一次 MCP Server，Host 方实现一次 Client，两边各自独立演进。

## 实践步骤

1. **收敛能力**：先把要暴露的操作列出来，砍到少量高内聚的 tool，写清 `name` / `description` / `inputSchema`；
2. **本地跑通**：用官方 SDK（Python / TypeScript）搭一个最小 server，走 stdio，先在本机验证工具能被列出、能被调用；
3. **接入 Host**：在 Agent 侧注册 server 配置，重启后确认工具列表正确出现，手动触发一次真实调用；
4. **按需暴露上下文**：适合的只读数据走 `resources`，固定的操作模板走 `prompts`，不要全塞成 tool；
5. **再上远程**：需要团队共享时切 Streamable HTTP，这一步才补鉴权和部署。

## 踩坑点

- **description 写得含糊**：它是给模型看的文档。没写清"何时用、何时不用"，模型就会选错工具或瞎填参数；
- **工具数量爆炸**：几十个 tool schema 全量注入上下文，token 成本和选择准确率一起恶化。按领域拆 server，或做按场景过滤；
- **stdio 在 Windows 上翻车**：命令路径、编码、`npx.cmd` 与 `npx` 的区别，都是高频问题；
- **schema 漂移**：改了 server 逻辑忘了同步 `inputSchema`，调用静默失败。对 schema 写断言测试；
- **安全边界**：敏感操作直接暴露成 tool 而没有确认机制；tool 返回内容可能携带注入文本，Host 侧要做隔离和审计；
- **协议版本不匹配**：握手失败先看 server 日志，别急着重装。

## 可复用建议

- 一个 server 只管一个领域，工具保持正交、避免重叠；
- 返回结构化且精简的结果，大输出落盘后只给路径引用；
- 危险操作加只读模式或显式确认参数；
- description 用固定模板写：用途 + 反例 + 参数示例；
- 把 MCP Server 当独立服务维护：有版本号、有 changelog、有最小集成测试。

## 总结

MCP 本质上是 LLM 时代的 LSP：不神奇，但把"每个工具写一遍胶水代码"变成"实现一次标准接口"。对做 Agent、插件和自动化的团队来说，越早把内部工具收拢成 MCP Server，后面换模型、换框架的迁移成本就越低。建议从一个小而痛的工具开始验证，再逐步迁移。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/29d76638b760fcda.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/310084744c803c43.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/6da6d8c244af2a60.png)

