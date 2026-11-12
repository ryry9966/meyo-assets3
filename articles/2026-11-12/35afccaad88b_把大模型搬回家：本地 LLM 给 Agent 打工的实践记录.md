---
title: 把大模型搬回家：本地 LLM 给 Agent 打工的实践记录
feedId: 44188
source: 综合讨论
publishedAt: 2026-11-12
---

## 背景

OpenClaw 的典型用法是让 Agent 读本地文件、调 MCP 工具、跑自动化脚本。这类任务天然贴着隐私数据——日志、代码库、聊天记录，把上下文整段发给云端 API 总归不踏实；而且 Agent 循环动辄几十轮调用，token 账单也不好看。于是把主力模型换成本地部署，云端只做兜底，本文记录整个过程。

## 问题

本地跑模型不难，难的是"给 Agent 跑"。三个硬指标：

1. **工具调用成功率**：小模型经常把参数 JSON 直接吐在 content 里，而不是结构化的 tool_calls 字段，Agent 链路直接断；
2. **吞吐**：一轮循环要消化几千 token 的工具返回，低于 20 tok/s 体感很差；
3. **显存/内存预算**：家用机通常只有 8–24GB 显存，或者 Apple Silicon 的统一内存。

## 做法

**硬件参考**：8GB 显存跑 7B Q4；12GB 跑 14B；24GB 显存或 32GB 统一内存可上 32B Q4。纯 CPU + 16GB 内存能跑 7B/8B，但吞吐一般只有个位数，只适合批处理。

**步骤**：

1. **运行时选型**：图省事用 Ollama；要精细控制用 llama.cpp 的 llama-server，两者都提供 OpenAI 兼容端点。
2. **模型选型**：中文 + 工具调用优先 Qwen 系列（如 qwen3:8b / 14b），显存紧张再降档。别只看榜单，直接测 tool call。
3. **关键参数**：上下文必须开够。Ollama 默认上下文很短（早期 2048，新版也常见 4k），llama.cpp 要显式 `-c 32768`，同时留意 KV cache 的显存占用。
4. **接入 Agent**：OpenClaw 侧新增一个 provider，base_url 指向 `http://localhost:11434/v1`（或 llama-server 的 8080），模型名对上即可。
5. **验证**：写一个最小脚本，注册一个假的 MCP 工具（比如"读文件"），连续跑 20 次，统计 tool_calls 解析成功率和端到端耗时。

## 踩坑点

- **chat template 与 tool-call parser 不匹配**：模型把 JSON 当正文输出。llama.cpp 需要加 `--jinja` 并选对对应 parser；
- **思考型模型输出 think 标签**：部分客户端解析不了，注意开关和清洗逻辑；
- **Ollama 默认几分钟就卸载模型**：Agent 间歇性调用会撞上冷启动，`keep_alive` 调大；
- **显存不够时 GPU offload 只卸一部分层**：速度反而不如纯 CPU 稳定，必须实测，别按层数想当然；
- **并发场景 KV cache 翻倍**：8GB 卡跑 14B，一开并行会话就 OOM；
- **量化别低于 Q4_K_M**：Q3 在结构化输出上的错误率上升非常明显。

## 可复用建议

- **用自己的 Agent 循环做基准**：核心指标是工具调用成功率和 tok/s，不是通用榜单分数；
- **本地/云端分层路由**：总结、格式化、批量打标走本地；复杂多步规划走云端，模型配置里留两条 profile 随时切换；
- **版本锁死**：模型 hash、运行时版本、启动参数进 git，出问题能回滚；
- **embedding 一并本地化**（如 bge-m3），RAG 链路就完全不出网了。

## 总结

本地 LLM 已经够格承担 Agent 日常工作的六七成：读文件、整理归纳、简单工具链都稳，复杂规划仍建议留给更强的云端模型。对 OpenClaw 用户来说，最有价值的第一个动作是写那个 20 次工具调用的小测试——十分钟就能判断你的机器 + 模型组合能不能上生产，比看任何评测都实在。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/4428bc860f997e3f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/742098aa475e0121.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/94cc3c242f720ed5.png)

