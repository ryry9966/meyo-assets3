---
title: OpenClaw 多模型路由实战：GPT 与本地模型的分工边界
feedId: 43661
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

跑 OpenClaw 的 agent 工作流一段时间后，大多数人都会撞到同一个问题：模型调用占了成本的绝大部分。我们团队的路径很典型——一开始全部走 GPT，月底账单难看；后来一刀切到本地模型，结果 agent 的多步规划和 MCP 工具调用质量明显下滑，失败率翻倍。多模型路由要解决的不是"选哪个模型"，而是"哪类任务配哪档能力"。

## 问题

静态切换之所以不行，是因为 agent 任务的异质性很强：

- 结构化抽取、摘要、分类：本地 7B/14B 模型完全够用；
- 多步规划、复杂工具编排：GPT 级模型的指令遵循和纠错能力明显更稳；
- 含敏感数据的 MCP 工具返回结果：按合规要求不应该出网。

规则一旦写死，任务形态一变就失效，所以路由必须是显式声明、可演化的配置，而不是散落在代码里的 if-else。

## 做法

1. **给任务打标签**。在 OpenClaw 的 pipeline 中给每个节点声明 `task_type`：`plan` / `tool_call` / `extract` / `summarize` / `chat`。这是路由的地基。
2. **建两个模型池**。云端池（GPT-4o 级别）和本地池（vLLM 部署 Qwen 系列，内网 GPU）。
3. **写路由规则**：

```yaml
routing:
  - match: { task_type: [plan, tool_call] }
    use: cloud
  - match: { task_type: [extract, summarize], tokens_lt: 6000 }
    use: local
  - match: { privacy: true }
    use: local
    fallback: none
```

4. **配 fallback 链**。云端失败降级到本地并打上"低置信"标记；隐私路径禁止回退云端；失败计数超阈值直接熔断告警。
5. **建评测闭环**。抽 50 条真实任务做回归集，每次改路由规则跑一遍，记录成本、P95 延迟和任务成功率。没有这个闭环，路由调优就是玄学。

## 踩坑点

- **本地模型的 function calling 和 JSON 遵循度普遍差**。涉及 MCP 工具调用的节点不要路由到本地，除非开启结构化输出模式并实测通过。
- **上下文长度错配**。本地 8k 窗口遇到长文档会静默截断，规则里必须加 token 上限判断。
- **GPU 冷启动**。本地服务空闲几分钟后首请求 10 秒以上，agent 侧的体感是"卡死"。要么保活，要么预热。
- **fallback 抖动**。云端限流时请求全涌向本地，把本地也打挂，务必给每条路径设并发上限。
- **重试导致成本虚高**。统计按"最终成功调用"口径算，否则优化效果会失真。

## 可复用建议

- 先分三类就够：规划/工具类走云端，抽取/摘要类走本地，敏感数据强制本地，其余默认云端。别一上来写二十条规则。
- 一个实用的判断原则：**这个任务的错误能否被下游步骤纠正？** 能纠正的用便宜模型，不能的用贵的。
- 路由规则当配置代码管理，变更走 review，必须附评测结果。
- 日志里记录每次路由决策的原因，排查质量问题时这是第一现场。

## 总结

多模型路由不是省钱技巧，而是把"任务类型—能力档位"的映射工程化。在 OpenClaw 里把 `task_type`、`privacy`、tokens 三个维度做实，配上合理的 fallback 和回归评测，大多数团队能在成本降低一半以上的同时，守住 agent 的关键质量线。路由规则会随模型迭代持续调整，把它当作长期维护的基础设施，而不是一次性优化。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/ba41f0f26ad63f7c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/9b7d9c42fa5ff8b0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/a5b60417a805d0bc.png)

