---
title: MCP 协议入门：Model Context Protocol 到底解决了什么问题
feedId: 43497
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

Function calling 出现后，Agent 终于能“动手”了，但接工具依然是脏活：每个客户端（桌面助手、自研 Agent 框架、IDE 插件）对接每个数据源（数据库、文件系统、内部 API）都得写一遍胶水代码。M 个应用 × N 个工具，就是 M×N 套集成，而且彼此不通用。

## MCP 解决的是什么

2024 年底 Anthropic 开源了 Model Context Protocol，一句话概括：把“应用接工具”标准化成一层协议。架构上是三件套：

- **Host**：跑模型的宿主，比如桌面客户端或你的 Agent 运行时
- **MCP Client**：Host 内嵌的协议客户端，负责握手与转发
- **MCP Server**：按协议暴露 tools / resources / prompts 的独立进程

M×N 就变成了 M+N：工具方写一次 Server，应用方实现一次 Client，其余靠握手协商。底层是 JSON-RPC 2.0，常见传输两种：本地 stdio、远程 Streamable HTTP。

也要说清边界：MCP **不**提升模型调用工具的准确率，也**不**替你解决所有 SaaS 的鉴权——它统一的是“发现、描述、调用”这条管道。

## 动手步骤

以“让 Agent 查内部 Git 仓库”为例：

1. 先找现成 Server（Git、filesystem 等都有官方实现），别急着造轮子
2. 用 Inspector 本地调试：`npx @modelcontextprotocol/inspector <server>`，可以可视化列出工具并手动调用
3. 在 Host 配置里注册，stdio 模式注意 command / args / cwd 三项
4. 在 Agent 里跑最小闭环：列出工具 → 选一个 → 调用 → 检查返回结构
5. 自建 Server 时，先定 tool 的 name / description / inputSchema，再写 handler

## 踩坑点

- stdio Server 往 stdout 打日志会污染 JSON-RPC 流，导致解析报错——日志一律走 stderr
- 工具 description 写得含糊，模型就会乱调；schema 就是给模型看的“文档”，值得认真写
- 返回值太大直接撑爆上下文，务必做分页、过滤、字段裁剪
- 第三方 Server 的工具描述本身可能被注入恶意指令，不要裸接，收紧文件系统与网络权限范围
- 忽略 initialize 阶段的协议版本协商，升级后常出现“工具列表为空”这类玄学问题
- 长耗时任务要支持 progress 通知或异步返回，否则调用端容易超时

## 可复用建议

- 一个 Server 只做一个领域；相关操作合并成带参数的工具，比堆几十个碎工具调用准确率高
- 把 tool schema 当 API 契约维护，加版本号，别随意改字段语义
- 团队内建一份“已审计 Server 清单”，统一入口、统一鉴权策略
- 给每次工具调用记录（截断后的）入参出参，排查问题时效率翻倍

## 总结

MCP 本质是一条管线标准：单 Agent 配单工具的场景没必要上；但当你有两个以上 Agent、或工具需要跨应用复用时，它的价值才开始兑现。工程上的正确姿势是——先复用生态，再按需自建，并把权限控制和调用日志当作一等公民来设计。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/1571c82cf14d6e4e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/1609b95e40720164.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/02a9ca7aa0dbe0e6.png)

