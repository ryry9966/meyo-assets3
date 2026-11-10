---
title: 本地 LLM 部署：在家用电脑上给 Agent 换一颗不联网的脑子
feedId: 44024
source: 综合讨论
publishedAt: 2026-11-10
---

# 背景

OpenClaw 的价值在于自动化链路全在自己机器上流转：消息、文件、脚本、定时任务。但模型如果还走云端 API，隐私和离线可用始终是短板。我最近花了两周把 Ollama 接进 OpenClaw，让模型也留在本地，这篇记录关键决策和踩过的坑。

# 问题：能聊天 ≠ 能当 agent 的脑子

先说核心观察：本地模型"能对话"和"能驱动 agent"差距很大，主要卡在三件事：

1. **上下文长度**。agent 的 system prompt、工具 schema、多轮历史叠加起来轻松上万 token，而很多本地运行时默认上下文只有 2k~4k。
2. **工具调用可靠性**。需要模型原生支持 function calling，且量化不能太狠，否则输出解析频繁失败。
3. **吞吐**。agent 是多步循环：思考、调工具、观察、再思考。5 步任务就是 5 次生成，每秒 10 token 的模型体验接近卡死，40 token/s 以上才算可用。

# 做法

**1. 先算显存账。** 粗略公式是权重加 KV cache：Q4_K_M 量化下 7B 约 4.5GB、14B 约 9GB、32B 约 19GB，32k 上下文的 KV cache 还要再加几 GB。8GB 显存卡的现实选择是 7B~14B，32B 别硬上。

**2. 选运行时。** Ollama 起步成本最低：`ollama serve` 自带 OpenAI 兼容端点，OpenClaw 侧把 base URL 指过去、填上模型名就能跑。llama.cpp server 控制粒度更细，vLLM 吞吐更高但更吃显存，按需再换。

**3. 选模型。** 优先原生支持工具调用、上下文 ≥32k 的 instruct 模型（Qwen 系列、Llama 3.1+ 等），量化从 Q4_K_M 起步，不要低于 Q4。

**4. 显式配置上下文。** 这是全文最重要的一步：Ollama 默认 `num_ctx` 很小，必须显式调到 16384 或更高，否则工具 schema 会被截断，表现为模型"忘了"自己有哪些工具。

**5. 用真实任务验收。** 别拿"你好"测。跑一条真实的多步 agent 流程，记录端到端耗时和失败率，再决定是否升级模型。

# 踩坑点

- **工具时灵时不灵**：排查半天发现是默认上下文太小，system prompt 被截断，调大 `num_ctx` 后恢复。
- **KV cache 撑爆显存**：权重放得下不等于长上下文放得下，OOM 时先砍上下文，而不是换模型。
- **空闲卸载**：Ollama 默认几分钟 unload，下次请求冷启动十几秒，设 `keep_alive` 保常驻。
- **并发排队**：定时任务和人工对话同时进来会排队，用 `OLLAMA_NUM_PARALLEL` 控制槽位，或把重任务错峰跑。
- **工具调用格式不匹配**：个别运行时的 function calling 模板和模型对不上，换运行时或换模型比改 prompt 有效。

# 可复用建议

沉淀了一个"新模型四步验机"流程，每次换模型先跑一遍再接入 OpenClaw：

1. 目标上下文长度下正常加载；
2. 探针 prompt 验证能正确发起一次工具调用；
3. 真实流程测端到端耗时；
4. 连续跑 30 分钟确认显存稳定。

另外两点：模型版本要钉死，同名模型更新后行为可能变化；把显存预算公式写进配置注释里，换卡换模型直接套。

# 总结

家用硬件跑本地大模型在 2025 年已经可行，但接进 agent 框架要按 agent 的约束设计：上下文显式配置、工具调用实测验证、吞吐用真实流程量。我的建议路径是 14B Q4_K_M 起步，链路跑通后再按显存余量升级。本地模型不是云端的平替，而是约束不同、但完全够用的工程选项。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/e66d89c24ad9e727.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/f65de8ebdde09e04.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/1b9ac3065c17b5bb.png)

