---
title: memory_recall 实测：语义搜索 vs 全文搜索，别二选一
feedId: 43476
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

在 OpenClaw 的日常使用里，`memory_recall` 是 agent 查自己历史记忆的入口：项目决策、踩坑记录、环境配置都以 markdown 条目存在本地记忆库，agent 靠这个工具（内置或通过 MCP 暴露均可）在对话中把它们捞回来。检索质量直接决定 agent 是"记得事"，还是一本正经地编。

## 问题

我的记忆库最初试过两个极端：

- **纯全文搜索**（SQLite FTS5）：查 error code、文件路径、命令名又快又准。但换个说法就搜不到——"上次那个数据库连接超时"捞不回写着 `psql: connection timed out` 的记录。
- **纯语义搜索**（本地 embedding + 向量检索）：近义改写能命中，精确词反而糊——搜 `uv sync --frozen`，回来一堆"关于 Python 依赖管理"的泛化内容。

哪个好？跑了个对照实验才知道，答案是"各管一段"。

## 做法

1. 构造 60 条真实查询做测试集（一半精确标识符，一半口语化改写），人工标注期望命中的条目。
2. 两套检索各跑一遍，记录 hit@3 和 P95 延迟。FTS5 本地查询 P95 < 5ms；本地 bge-small + 暴力向量检索 P95 约 30ms，都在可接受范围。
3. 结果：精确词查询 FTS5 命中约九成，语义只有六成多；口语化改写反过来，语义八成，全文不到四成。
4. 结论：**混合检索**。全文和向量各取 top-10，用 RRF（倒数排名融合）合并取 top-5 喂给 agent。融合后两类查询都稳定在九成上下，代价只是十几行代码。

## 踩坑点

- FTS5 默认 unicode61 分词器对中文几乎不可用（整句当一个 token），要么上 trigram，要么接 jieba 分词。中文记忆库这步躲不掉。
- 通用 embedding 模型对代码片段、error code 很弱，精确标识符别指望语义召回兜底。
- 换 embedding 模型 = 全库重建向量索引。模型名必须写进记忆库元数据，否则新旧向量混存，结果悄悄劣化还查不出原因。
- 失效记忆（过期配置、废弃决策）会持续污染 top 结果，recall 侧加时间衰减比直接删除稳妥。

## 可复用建议

- 记忆条数在几千条以内时，别急着上向量库，FTS5 + 好的分词就够。
- **先建评测集再选型**。60 条查询的工作量不大，收益远大于看十篇博客。
- `memory_recall` 返回值里带上命中来源（全文/语义）和分数，agent 判断可信度、你排查 badcase 都用得上。
- 把没命中的查询记日志，定期回填测试集，评测集会越来越接近真实分布。

## 总结

语义搜索和全文搜索不是竞品，是分工：全文管精确标识符，语义管自然语言改写，RRF 负责缝合。在 OpenClaw 这种本地优先的场景下，SQLite FTS5 + 本地小模型 embedding 的组合，延迟、成本、隐私都能打。先测再选，别凭感觉。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/2320eb87fafa6626.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/eecfdd21bf9e7a9e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/5f1ac8bb3dc9fb81.png)

