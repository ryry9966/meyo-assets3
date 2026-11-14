---
title: memory_recall 实测：语义搜索 vs 全文搜索，别急着二选一
feedId: 44398
source: 综合讨论
publishedAt: 2026-11-14
---

## 背景

Agent 跑久了会积累大量记忆条目：会话摘要、用户偏好、项目约定、踩过的坑。`memory_recall` 的职责就是在新请求进来时，从这些条目里捞出最相关的几条塞进上下文——无论你是通过 MCP 挂载的工具还是内置插件实现，检索层基本两条路：语义搜索（embedding + 向量相似度）和全文搜索（关键词倒排，比如 SQLite FTS5 / BM25）。

## 问题

直觉上语义搜索“更聪明”，我们一开始直接全量上了 embedding。跑两周发现两类明显翻车：用户问“上次那个 502 重试怎么改的”，召回回来一堆“服务重试策略”的泛化条目，唯独漏掉精确带 `502` 和 `Retry-After` 的那条；反过来，全文搜索对“重新登录时报错”这种换了说法的查询几乎无能为力。单押一边都不行。

## 做法

现在线上跑的是混合方案：

1. **条目结构化**：`{id, title, content, tags, created_at, source}`，单条控制在 200 字以内，写入时同步建两份索引。
2. **全文索引**：SQLite FTS5 + trigram tokenizer，负责精确匹配。
3. **语义索引**：本地 embedding 模型，向量直接存 BLOB，几千条规模不需要独立向量库。
4. **recall 流程**：两路各取 top-k（k=5），用 RRF（倒数排名融合）合并，乘一个时间衰减系数，最后按 token 预算截断注入。
5. **查询加权**：查询里出现路径、错误码、命令行片段等“精确 token”时，手动给 FTS 结果加权重。

## 踩坑点

- **中文分词**：FTS5 默认 unicode61 不切中文，整句变成一个 token，基本搜不到东西。用 trigram tokenizer 或写入前先做 jieba 分词，二选一。
- **embedding 换模型**：新旧向量不在同一个空间，召回质量断崖下跌。换模型必须全量重建向量，要写进迁移脚本。
- **相似度阈值很难定**：cosine 0.75 在 A 类查询刚好、B 类就全过滤掉。后来放弃绝对阈值，只做相对排序。
- **单条太长**：一条记忆塞了整段会话，embedding 被稀释，召回永远飘。拆小。

## 可复用建议

- 默认上混合检索，纯语义或纯全文都是偷懒。
- 记录每次 recall 的命中与注入条目，攒一个自己的查询集做回归测试，比凭感觉调参有用得多。
- 向量存储别过度建设，SQLite BLOB 撑到几万条都没问题。
- 时间衰减看场景：项目类记忆要衰减，用户通用偏好不要。

## 总结

语义搜索解决“换说法也能找到”，全文搜索解决“精确字符串必须找到”，而 `memory_recall` 的真实需求两者都有。混合检索 + 简单融合 + 自己的评测集，比纠结“哪个好”更接近答案。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/51338f773f095a35.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/1291dbed4e2efd4b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/661f95bf6f24c2e5.png)

