---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 44251
source: 综合讨论
publishedAt: 2026-11-12
---

跑 OpenClaw 一段时间后，最常见的瓶颈往往不是模型能力，而是 system prompt 越写越长：工作流约定、内部工具用法、部署检查清单……全塞进提示词，上下文被固定吃掉一大截，任务无关时还稀释模型对关键指令的注意力。Skills 机制就是针对这个问题的解法：把能力拆成独立目录，平时只在上下文里保留一行元数据，命中时才加载正文。

## 问题

没有 Skills 时通常两条路。一是全写进系统提示词：token 成本是固定支出，十几个流程全量常驻，大多数轮次根本用不上；长提示词还会让指令互相干扰，模型"看过但没照做"的概率明显上升。二是靠模型参数记忆：私有流程它根本不知道，知道的多半也是过时版本。本质上我们需要的是**按需注入**。

## 做法

Skill 就是一个约定格式的文件夹，以用户级技能为例：

```text
~/.openclaw/skills/
  deploy-check/
    SKILL.md
    scripts/precheck.sh
  weekly-report/
    SKILL.md
    reference/metrics.md
```

关键在 SKILL.md 的两段结构：

1. **frontmatter**：`name` 和 `description`（还可声明所需工具或命令）。每轮对话里 agent 只能看到所有技能的 name + description，靠 description 决定要不要读正文。所以要把它写成路由规则——"Use when…" 加具体动作词和用户会提到的关键词，而不是一句功能简介。
2. **正文**：命中后才整体进上下文。控制在几百行内，长细节（参数表、完整配置、历史坑）拆到 reference 文件，正文只写"什么情况读哪个文件"，让第二次加载也是按需的。

放置位置决定优先级：workspace 内的 `skills/` 优先级最高，适合项目私有流程；`~/.openclaw/skills/` 放个人通用能力。修改 SKILL.md 后需要新开会话才生效——元数据是随 system prompt 构建的，不会热更新。

验证方式很朴素：派一个对应任务，看 gateway 日志里有没有技能加载记录；再故意派一个不该触发的任务，确认没有误加载。成本上，每条元数据大约几十 token，正文触发一次在 1–3k token 量级，日志里都能对上。

## 踩坑点

- description 太模糊 → 永远不触发；太宽 → 每轮都触发，等于变相全量注入。把它当分类器调，准备正反例测试。
- 把所有内容堆进正文：一旦触发，成本和巨型提示词没区别，渐进式披露就失效了。
- 技能名与内置技能撞名，排查时容易搞混加载来源，建议加个人前缀。
- 脚本里写绝对路径，换台机器就断；写成相对技能目录的引用，模型会自己拼。
- 别和 MCP 职责搞混：MCP 管"活的连接"（API、数据源），Skills 管"流程知识"（怎么用这些工具把事做对）。同一件事两边都写，迟早不同步。

## 可复用建议

- 一个技能一个能力，抵制"万能技能"的冲动。
- 把 skills 目录纳入 git，跨机器同步方便，回滚也有据可查。
- 技能不只是文档：配脚本让它变成确定性操作，模型只负责判断时机。
- description 里埋用户真实会说出的词，这比功能描述更能提高命中率。
- 定期清理：长期没触发的技能，description 大概率已经失效。

## 总结

Skills 的核心价值是渐进式披露：元数据常驻、正文按需、细节再按需。在上下文预算有限的前提下，这种分层加载比无限堆提示词可持续得多。建议从你最痛的一个重复流程开始写第一个技能，跑两周、看日志、再迭代。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/2462b24903ac9a58.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/107ee1e9a2fff98e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/7a42fedc77a720d0.png)

