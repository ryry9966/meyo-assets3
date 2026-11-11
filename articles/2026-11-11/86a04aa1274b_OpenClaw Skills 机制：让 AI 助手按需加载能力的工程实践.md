---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力的工程实践
feedId: 44127
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景

随着 Agent 接的能力越来越多，一个老问题会反复出现：把所有工具说明、操作手册全部塞进 system prompt，上下文很快失控。OpenClaw 的 Skills 机制给出的思路是**渐进式披露（progressive disclosure）**：常驻上下文的只有每个 skill 的名字和一句话描述；正文、模板、脚本只在被命中时才注入。

## 问题

全量加载的代价很具体：

- token 占用随 skill 数量线性增长，成本和延迟同步上升；
- 指令越多，模型越容易漏读、错读，触发准确率反而下降；
- 多个能力的触发词互相干扰，调试时说不清到底是哪段 prompt 生效。

## 做法

Skills 本质是一个约定好结构的目录：

```
skills/
  weekly-report/
    SKILL.md
    templates/report.md
    scripts/fetch_data.py
  data-clean/
    SKILL.md
```

`SKILL.md` 的 frontmatter 是触发的关键：

```yaml
---
name: weekly-report
description: 拉取仓库 commit 与 issue 数据生成团队周报。当用户要求"写周报/汇总本周进展"时使用。
---
```

运行时分三层加载：

1. **常驻层**：所有 skill 的 name + description，占用极小；
2. **命中层**：用户请求匹配到某个 description 后，才加载该 skill 的正文；
3. **资源层**：正文里引用的模板、脚本，按需再读、再执行。

落地步骤建议：建目录 → 写 frontmatter → 正文只写流程，细节外移到 templates/scripts → 用真实问法验证触发 → 统计单次会话的 token 增量。

## 踩坑点

1. **description 写得太虚**。比如"帮助用户处理各种任务"，等于没写，模型只能瞎猜。必须写清"做什么 + 何时用"。
2. **职责重叠导致歧义**。两个 skill 都能响应"整理数据"，模型会随机选。要么合并，要么在 description 里划清边界。
3. **把一切塞进正文**。命中一次就吃掉上千 token，渐进式披露形同虚设。正文控制在几百 token，长模板、长清单全部外移。
4. **脚本依赖不声明**。skill 在你机器上能跑，换台机器就挂。依赖和运行方式要写进正文。
5. **改完不回归**。修改 skill 后，旧会话/缓存里仍是旧行为，务必用同一组问法重测。

## 可复用建议

- **一个 skill 只做一件事**，粒度到"用户的一个明确意图"为止。
- 维护一份 **10 条左右的触发测试集**：正向问法应命中，近似但无关的问法不应命中，每次改动跑一遍。
- description 当成**检索文档**来写，而不是宣传文案。
- 给 skill 加版本号或 changelog，方便回滚和排查"行为变了"这类问题。

## 总结

Skills 机制的价值不在于"能力多"，而在于上下文里永远只有当下需要的那一部分。写 skill 的本质，是给模型写一份检索友好的文档：元数据负责被找到，正文负责被执行，资源负责按需取用。把这个分层想清楚，Agent 的能力扩张才不会以牺牲稳定性为代价。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/99f7323c4e2c5d80.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/28379f275656686a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/b85ba02b2040a591.png)

