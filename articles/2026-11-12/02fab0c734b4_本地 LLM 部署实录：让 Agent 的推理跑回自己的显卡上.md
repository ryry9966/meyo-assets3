---
title: 本地 LLM 部署实录：让 Agent 的推理跑回自己的显卡上
feedId: 44153
source: 综合讨论
publishedAt: 2026-11-12
---

## 背景

过去两年，我的推理负载基本全走云端 API。促使我认真做本地部署的是两件事：一是涉及私人文档和日志的 MCP 工具链，数据不想出本机；二是摘要、分类、embedding 这类高频小任务，按 token 计费算下来并不便宜。这篇帖记录我把 14B 模型跑在 12GB 显存家用机上、并接进现有 OpenClaw Agent 工作流的完整过程，重点讲决策依据，不复述安装教程。

## 先想清楚的问题

本地部署不是"越大越好"，动手前先回答三个问题：

1. 本地模型承接什么任务？兜底还是主力？
2. 显存 / 内存的真实上限是多少？
3. 工具调用在你的 Agent 链路里是不是硬需求？

第三个最关键。不少量化小模型写自然语言没问题，但输出严格 JSON、稳定调用工具的能力会明显退化——这直接决定它能不能进 MCP 链路。

## 部署步骤

1. **盘点硬件。** 粗估：Q4_K_M 量化下每 1B 参数约 0.6GB，再加 10–20% 的 KV cache 和运行时开销。12GB 显存跑 14B Q4 勉强够，7B–8B 很从容；无独显、16GB 内存的机器跑 7B Q4 可行，速度靠 CPU 硬扛。
2. **选运行时。** Ollama 上手最快、自带 OpenAI 兼容端点；llama.cpp 控制粒度最细；显存 24GB+ 且追求吞吐可以上 vLLM。我从 llama.cpp 起步，最后留在 Ollama，理由是生态和维护成本。
3. **选模型与量化。** 优先官方支持 tool calling 的 instruct 版本；量化 Q4_K_M 起步——Q8 的质量提升对 Agent 任务感知不大，显存翻倍是真痛。
4. **起服务，改两个参数。** Ollama 默认上下文只有 2048，Agent 多轮对话很快爆掉，用 Modelfile 或 API 参数显式拉到 8k–16k；`keep_alive` 设长一点，避免每次请求重载权重。
5. **接入 Agent 链路。** base_url 指向本机端点即可。我做了一层路由：本地承接高频低风险任务，复杂规划和工具调用仍走云端。OpenClaw 和 MCP 侧不用改配置，本地端点对上层是透明的。

## 踩坑点

- **显存估算漏了 KV cache。** 长上下文时 KV cache 能吃掉好几个 GB，"权重装下了"不代表跑得动，实际是部分计算溢出到内存，速度暴跌。
- **GPU 层数静默降级。** 显存不够时 llama.cpp 会把部分层放到 CPU，日志里利用率忽高忽低。跑一个 1k token 的 prompt 观察 tok/s 是否异常，别只盯着显存占用。
- **小模型的 tool calling 是玄学。** 同一个模型，system prompt 措辞一变，JSON 合规率能从 90% 掉到 60%。写回归用例，别靠感觉。
- **别用短 prompt 测性能。** prefill 和 decode 的速度差好几倍，Agent 场景 prefill 占大头，测试 prompt 至少 2k token 才有参考价值。
- **模型当依赖管理。** 给 GGUF 文件记 hash，换模型前跑一遍 Agent 回归集，否则"升级"会变成隐性 bug 源。

## 可复用建议

- 显存公式背下来：参数量 × 0.6GB（Q4）+ KV cache + 1–2GB 余量。
- 本地模型的定位是"高频兜底"，不是全面替代；先在一个明确的小任务域跑稳，再考虑扩。
- 吞吐和首 token 延迟分开记录，Agent 的体验主要由首 token 决定。
- 固化一份评测集：10 个工具调用用例 + 10 个纯文本用例，每次换模型或量化都跑一遍。

## 总结

本地部署的价值不在"跑起来"，而在于对自己的延迟、成本、数据边界有了确定性控制。对个人开发者来说，一台 12GB 显存的机器加一个 tool calling 靠谱的 8B 模型，已经能承担 Agent 链路里相当比例的日常推理；剩下的部分交给云端，混合路由是目前投入产出比最高的形态。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/39518f429862facd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/173549224b7000a2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/7c60ed896aae425e.png)

