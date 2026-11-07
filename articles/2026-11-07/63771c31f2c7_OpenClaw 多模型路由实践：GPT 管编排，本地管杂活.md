---
title: OpenClaw 多模型路由实践：GPT 管编排，本地管杂活
feedId: 43745
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

OpenClaw 跑了一段时间后，大多数人都会遇到同一个分叉口：所有会话都走 GPT，账单和隐私压力随使用量线性上涨；全切本地模型，工具调用和长上下文又频繁翻车。OpenClaw 网关本身支持多 provider 并存、按 agent 绑定模型和 fallback，所以难点从来不是“能不能配”，而是“按什么边界分”。

## 问题：一刀切都要付代价

**全走 GPT**：一是成本，批量摘要、字段抽取这类任务用旗舰模型纯属浪费；二是隐私，让包含聊天记录和本地文件的上下文默认出网，多数场景没有必要。

**全走本地**：7B~14B 档的模型在多轮工具编排、长 system prompt、结构化输出上明显不稳，而且翻车方式往往是静默的。

结论是路由必须做，但要做成工程化的静态分工，而不是每次靠手感切换。

## 做法：四步

1. **把 Ollama 挂成 OpenAI 兼容 provider**，与 OpenAI 并列：

```jsonc
// ~/.openclaw/openclaw.json（示意，字段名以你所用版本为准）
"models": { "providers": [
  { "id": "openai", "baseUrl": "https://api.openai.com/v1", "apiKey": "sk-***" },
  { "id": "ollama", "baseUrl": "http://127.0.0.1:11434/v1", "api": "openai-completions" }
]}
```

2. **主 agent 设 primary + fallback**：primary 用 GPT，fallback 指向本地大参数模型，链长不超过 2，专治限流和偶发超时。

3. **杂活下沉到子 agent**：新建“清洗/摘要”类子 agent，单独绑定本地模型（如 qwen3:14b），由主 agent 通过工具调用派发任务。上下文不出主会话，主会话模型也就不用降级。

4. **固定四维判断**：任务类型、上下文长度、是否需要工具调用、是否涉敏。四项里只要“需要工具”或“涉敏”命中，要么走 GPT，要么把任务改造成纯文本留在本地。

## 踩坑点

- **本地模型的 function calling 是重灾区**：失败常表现为“编参数”而不是报错。给本地 agent 只挂只读类工具，能规避大部分事故。
- **上下文窗口不一致**：128k 切到 32k 会静默截断，症状是“记忆丢失”。路由前先粗估 token 量。
- **风格漂移**：同一份 system prompt 换模型后语气和输出格式会变，本地模型需要单独调温度和提示词，别共用。
- **fallback 掩盖真实错误**：命中 fallback 时打日志，否则配额问题会被无限期吞掉。

## 可复用建议

- 攒 20 条典型任务做小评测集，换模型、改参数前跑一遍，用结果代替感觉。
- 本地选型顺序：先确认工具调用支持，再看上下文窗口，最后才看参数量。
- 成本按 agent 维度统计而不是按模型，才能验证路由是否真的省钱。
- 别指望网关做语义级自动路由。先把静态分工和命中日志做扎实，数据够了再谈自动化。

## 总结

多模型路由的价值不在省那点 API 费，而在让能力、隐私、成本三者对齐。用“能力边界”划出必须走 GPT 的最小集合，其余尽量下沉本地；规则写进配置和日志，而不是记在脑子里——这才算工程化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/af06ab0238fa4934.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/16c87ac723694c9e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/60575050c1287f30.png)

