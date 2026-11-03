---
title: Agent 记忆系统设计：怎么让 AI 助手真正记住你的偏好
feedId: 43378
source: 综合讨论
publishedAt: 2026-11-03
---

## 背景

LLM 无状态，每次会话都是陌生人。上下文窗口再大也装不下"这个用户喜欢什么"，对 OpenClaw 这类常驻 Agent，记忆系统是刚需。但多数实现停留在"把聊天记录塞进向量库"，实际效果一般。

## 问题

三种典型失败模式：

- **全量塞 system prompt**：token 成本飙升，关键偏好被无关内容稀释；
- **只做会话内记忆**：重启即失忆；
- **纯向量检索聊天记录**：召回大量噪声，新旧偏好同时命中，Agent 无所适从。

记忆的难点不在"存"，而在提取、检索与生命周期管理。

## 做法：分层记忆 + 写入检索闭环

先把记忆拆成四层，分别对待：

- **工作记忆**：当前会话上下文，不持久化；
- **情景记忆**：会话结束后的摘要存档，低频回溯时检索；
- **语义记忆**：从对话中提炼的结构化偏好（核心层）；
- **程序记忆**：固化成 Skill/Workflow，比记忆更硬。

语义记忆的落地分五步：

1. **定 schema，不存自由文本**。SQLite 一张表够用：`memories(id, type, content, tags, embedding, confidence, last_seen, status)`，type 区分 preference/fact/correction，status 用于标记 superseded。
2. **双入口写入**。会话结束跑离线提取 prompt，从 transcript 产出候选记忆；同时给 Agent 一个 `memory_write` MCP 工具，用户说"记住我喜欢 X"时直接写入。所有写入走同一 schema。
3. **混合检索注入**。会话开始先按 tags 做确定性过滤，再叠加向量相似度取 top-K（经验值 5–10 条），拼成紧凑的偏好块注入 system prompt，严格控制在几百 token 内。
4. **冲突不删除，做 supersede**。新记忆与旧记录语义冲突时，旧记录置为 superseded 并指向新记录；confidence 随重复确认上升，随长期未召回衰减。
5. **管理界面必须有**。哪怕只是 CLI 的 list/edit/delete，用户能看见 Agent 记了什么，才敢让它继续记。

## 踩坑点

- **别让模型自由写记忆**。没有 schema 约束时会写出"用户今天心情不错"这类垃圾，type 和 tags 必填。
- **纯向量检索有否定陷阱**："不要用 tabs"会和"用 tabs"召回在一起，检索层必须保留原文和标签，不能只依赖 embedding。
- **偏好会变**。三个月前提取的"用 Vue"可能已过时，last_seen 和 supersede 机制不可省。
- **注入要克制**。我早期注入过 30+ 条记忆，Agent 反而开始忽略它们；减到 8 条以内，关键行为明显变稳。
- **隐私默认最小化**。提取 prompt 里明确排除密钥、证件类内容，并提供一键 forget。

## 可复用建议

- 把记忆做成独立 MCP server（`memory_search` / `memory_write` / `memory_forget` 三个工具），与 Agent 主逻辑解耦，换模型不用重写。
- 记录每次注入了哪些记忆到哪个会话，排障时这是最有用的日志。
- 维护一个十几条的小评测集：每次改完检索策略，跑一遍"重启后是否仍遵循偏好"，比肉眼感受可靠。

## 总结

Agent 记忆本质是工程问题：schema 决定存什么，混合检索决定取什么，supersede 与衰减决定信什么，透明管理决定用户敢不敢用。从 SQLite 加标签过滤起步就够了，先跑通闭环，再谈优化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-03/1938c539716450f7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-03/4e4f5346251b5884.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-03/f6200e36663e0663.png)

