---
title: memory_recall 实战：语义搜索 vs 全文搜索，我的选型和踩坑
feedId: 43646
source: 综合讨论
publishedAt: 2026-11-06
---

## 背景

OpenClaw 的 agent 跑久了，靠 memory_recall 把历史对话、决策记录、配置笔记捞回上下文。检索层常见的两条路线：语义搜索（embedding + 向量库）和全文搜索（FTS5 / BM25 / ripgrep）。社区里经常争论哪个“更先进”，我用真实会话数据跑了两个多月，说说工程上的结论。

## 问题

- **语义搜索**：问“上次部署超时怎么解决的”能召回，但搜 `ERR-4021`、`config.yaml`、一段代码经常漏——错误码、路径、精确 token 这类东西，embedding 相似度天然不可靠。
- **全文搜索**：精确命中很强，但换个说法就找不到。当时记的是 "postgres connection refused"，现在问“数据库连不上”，一条都搜不到。
- 单选任何一个，都有固定盲区。盲区落在哪，取决于你的记忆里是“想法”多还是“代码片段/ID”多。

## 做法

我的方案是**混合召回**，步骤如下：

1. 记忆落库时同时写三样：原文全文、标签 + 时间戳、embedding（中英混合场景用多语模型如 bge-m3）。
2. 全文层用 SQLite FTS5，中文需要 jieba 分词或 trigram，够用且零运维。
3. recall 时两边各取 top 10，用 RRF（reciprocal rank fusion）合并排名，再叠加 `时间衰减 × 0.8 + 相似度` 的权重。
4. 合并结果交给 LLM 做一次轻量筛选再进上下文，不要整包塞入。
5. 建一个 20~30 条的回测 query 集（从自己历史会话里抄真实问题），每次改检索参数跑一遍，防止“感觉变好了”。

## 踩坑点

- 一开始用了以英文为主的 embedding 模型，中文 query 召回率暴跌，换多语模型才稳。
- chunk 切太大（>500 字），一条记忆混了三件事，召回即噪音；**一事一块**。
- 只按相似度排序，三个月前的旧记忆永远压不住最近写入，必须加时间衰减。
- 写入时同步算 embedding，高峰期明显卡顿；改成异步队列补算。
- FTS5 默认分词器对中文基本失效，别裸用。

## 可复用建议

- **先全文后语义**：记忆量 < 5000 条时，纯 FTS5 + 好的查询改写可能就够了，别一上来就上向量库。
- 混合是默认答案，但一定保留 raw text，索引随时可重建，embedding 模型可随时替换。
- 回测集是护身符：没有 20 条真实 query，任何检索调参都是玄学。

## 总结

语义搜索负责“意思相近”，全文搜索负责“一字不差”，memory_recall 最好两个都要。混合召回 + 时间衰减 + 固定回测集，是我目前最省心的组合。别追“更先进”，先看你的记忆库里到底是代码片段多，还是想法多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/77a736dd09485f84.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/8f32380857a7e71a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/f55464d88c3f0bbd.png)

