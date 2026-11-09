---
title: OpenClaw 多模型路由实战：GPT 与本地模型的分工边界
feedId: 43938
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

OpenClaw 的模型层一直是开放的：`openclaw.json` 里可以同时挂云端 provider（GPT 系列、Claude 等）和本地推理端（Ollama、LM Studio、vLLM）。但不少人的配置是“一刀切”——要么全走旗舰模型，要么为了省钱全走本地。前者月底账单难看，后者在多步工具调用场景里成功率明显下降。这篇整理一下社区里几个人跑了几个月后的路由方案，供参考。

## 问题

核心矛盾在于：agent 任务的质量方差远大于普通对话。同样是“总结这条消息”，本地 7B 和旗舰模型差距不大；但“读两个 MCP 工具的返回、规划下一步、再调用工具”这种链路，量化后的本地模型经常在 JSON 参数格式上翻车，一环失败整条链重跑，时间成本反而更高。

## 做法

我们把任务分成三档：

1. **规划与复杂工具链**（planning、多工具编排、长上下文推理）→ 云端旗舰模型，作为 primary；
2. **轻量收尾**（摘要、改写、分类、格式化输出）→ 本地 7B~14B，Q4 量化即可；
3. **Embedding 与记忆检索** → 一律本地，避免敏感数据出网。

配置上利用 OpenClaw 的 per-agent model 加 fallback 机制，结构大致是：

```json
{
  "agents": {
    "main": { "model": "openai/gpt-4.1", "fallbacks": ["ollama/qwen2.5:14b"] },
    "summarizer": { "model": "ollama/qwen2.5:7b" }
  }
}
```

再配合一个路由 skill：入口先用本地小模型做一次分类（判断任务类型、预估 token 数），命中“规划类”或超过阈值就转发云端。注意分类这一步本身必须用本地模型，否则路由开销会吃掉省下来的成本。

## 踩坑点

- **function calling 质量**：本地模型量化后 tool call 参数经常缺字段或类型漂移。建议本地档只接“返回纯文本”的工具，涉及 MCP 结构化调用的留给云端。
- **上下文截断**：本地 8k 模型接到长会话会被静默截断，工具返回一大就崩。本地档上游加一层摘要压缩更稳。
- **静默降级**：fallback 触发时日志不一定明显，排障第一步是确认 session log 里实际用的 provider，别对着错误配置调半天。
- **本地端点升级**：Ollama 拉了新版本后行为漂移。生产环境固定 tag（如 `qwen2.5:14b-instruct-q4_K_M`），别用 `:latest`。
- **参数语义差异**：不同 provider 对 temperature、max_tokens 的解释不完全一致，同一套 agent 参数别原样复制到本地端点。

## 可复用建议

- 先跑两周“全云端”作为基线，记录每类任务的失败率和 token 消耗，再逐档下沉到本地，用数据决定边界，而不是凭感觉。
- 路由规则宁少勿多，两三条清晰的规则好过十条件件判断。
- 保留手动覆盖入口（会话里直接切模型的命令），出问题时能立刻切换对比，定位是模型问题还是编排问题。

## 总结

多模型路由不是省钱技巧，而是给不同质量要求的任务匹配不同的算力。我们的经验值是：80% 的日常消息量可以走本地，但决定 agent 成败的那 20% 关键调用，别省。先量化、再分级、小步下沉，比一步到位的“完美路由方案”靠谱得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/d3491baee422ae83.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/fa6b25eed8cb7eee.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/a7037aeb23497bc1.png)

