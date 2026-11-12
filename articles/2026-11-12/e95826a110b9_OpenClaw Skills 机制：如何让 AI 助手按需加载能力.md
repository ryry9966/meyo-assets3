---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 44243
source: 综合讨论
publishedAt: 2026-11-12
---

## 背景：上下文不是免费的

做 Agent 时间长了都会撞上同一堵墙：system prompt 越来越肥。早期只有两三个工具、一份流程说明还好；后来接了消息通道、加了发布规范、补了排障手册，全塞进常驻上下文。结果每次会话都在为"大概率用不到"的内容付 token，更糟的是注意力被稀释——模型反而容易漏掉本轮真正要紧的那条指令。

OpenClaw 的 Skills 机制就是针对这个问题的解法：把能力拆成独立的 Markdown 单元，不常驻，按需加载。

## 工作原理：三层渐进式披露

Skills 的核心设计是 progressive disclosure，分三层：

1. **元数据层**：会话启动时，OpenClaw 只把每个 skill 的 `name` 和 `description`（SKILL.md frontmatter）注入上下文，装几十个 skill，常驻成本也只有几百 token；
2. **正文层**：模型根据 description 判断当前任务是否命中某个 skill，命中才用读文件工具加载完整的 SKILL.md；
3. **资源层**：SKILL.md 还可以引用更多文件（参考文档、示例、脚本），真正需要时才读。

能力清单常驻、能力细节按需，这就是"按需加载"的全部机制。

## 亲手写一个 skill

以"发布前检查"为例，五步：

1. 在 `~/.openclaw/skills/` 下建目录 `release-check/`，放入 `SKILL.md`；
2. 写 frontmatter：`name` 用小写短横线；`description` 是关键，写给模型看，讲清"什么场景该用我"，比如"当用户要求发布、上线、发版前检查时执行本流程"；
3. 正文只写核心步骤、命令和约定，控制在 150 行内；细节（如回滚预案）拆到 `references/rollback.md`，正文注明"何时读哪个文件"；
4. 确定性操作包成脚本（如 `scripts/precheck.sh`），让模型直接执行，而不是即兴发挥；
5. 重启会话或用 `openclaw skills list` 确认注册，然后问一句"帮我发个版"，观察是否正确命中并加载。

## 踩坑点

- **description 写给人看**："数据处理工具"这种简介永远不会被触发，要写触发条件和关键词；
- **一个 skill 干多件事**：选择时产生歧义，两个 skill 描述重叠时模型经常选错，拆开；
- **正文贪多**：塞进 2000 行，加载一次的成本就回到老路，等于白拆；
- **脚本依赖没交代**：需要虚拟环境或某个 CLI 却不写明，模型一跑就报错；
- **只在本地验证**：网关部署在服务器上时路径和环境不同，务必在真实运行时过一遍。

## 可复用建议

- 一 skill 一职责，宁可多拆几个；
- description 套模板：适用场景 + 关键词 + 明确不适用的情况；
- 三层做内容预算：元数据一两句、正文 150 行内、资源层不限；
- 高频且确定性强的操作固化成脚本，skill 只当"说明书 + 入口"；
- 用 git 管理 skills，description 的改动本质是在改"召回率"，留 10 条典型问题做评测集，改完跑一遍再合入；
- 先去 clawhub 逛逛别人的 SKILL.md 写法，多数常见场景不必从零造。

## 总结

Skills 的本质是把"能力说明书"从上下文常驻改成按需调用，成本从 O(全部能力) 降到 O(本轮用到的能力)。整个机制里最值得花时间的是 description，它相当于模型世界里的函数签名，签名写不好，函数再好也调不到。建议从 system prompt 里最稳定的一段流程开始抽离，一次一个，收益立竿见影。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/07569d2ecf210988.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/a50075082fd81287.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/0b0a0cef67f1a439.png)

