---
title: 把 Git 交给 Agent：提交与分支自动化的最小落地实践
feedId: 44220
source: 综合讨论
publishedAt: 2026-11-12
---

## 背景

日常开发里 Git 操作占比不小但价值密度低：切分支、暂存、写 commit message、推远端、开 PR。这些动作重复且规则明确，很适合交给常驻 AI 助手。OpenClaw 的定位就是本地常驻 Agent，天然适合接这类"手边杂活"。这篇记录我在自己仓库上落地的一个最小方案，目标不是无人值守，而是"你说一句，它做完，且不出事"。

## 问题

直接让 Agent 跑 git 命令会遇到几个真实问题：

1. **工作区脏**：Agent 分不清哪些改动是你有意暂存的，一梭子 `git add -A` 容易把调试文件、密钥一起提交。
2. **分支混乱**：不建分支直接在 main 上提交；或者建了分支，命名完全不可检索。
3. **危险操作**：`push --force`、`reset --hard`、`clean -fd`，一次误操作代价很高。
4. **commit 信息质量不稳定**：时而过详时而敷衍，历史没法读。

## 做法

方案分三层：

**第一层：工具收敛。** 不给裸 shell，而是包一层受限的 git 工具——OpenClaw 里可以做成一个 skill，内部就是几个白名单命令的封装：`status`、`diff`、`add`（指定路径）、`commit`、`branch`、`switch`、`push`（限当前分支）。危险命令根本不在工具面里，Agent 想调也调不到。如果你习惯 MCP，用一个 git MCP server 也是同样思路：收敛工具面。

**第二层：约定写进 skill 描述。** 在 skill 说明里固化仓库规范：

- 任何提交前先建分支，命名 `feat/xxx`、`fix/xxx`、`chore/xxx`；
- commit 遵循 Conventional Commits，正文写"为什么"，不写"改了什么"（diff 自己会说话）；
- 提交前先展示 `git status` 和 diff 摘要给我确认，除非我明确说"直接提交"。

**第三层：确认环。** 高风险动作（push、碰 main）一律停下等确认；低风险动作（建分支、本地 commit）放行。跑了两周后我把本地 commit 也设为默认放行，push 仍保留确认。

实际体感：小改动从"改完代码到推上去"大约从五分钟缩到一分钟，commit message 质量比我手写的还稳定——因为 Agent 会真的去读 diff。

## 踩坑点

- **别忽视 hook 输出**。仓库有 pre-commit 格式化钩子时，Agent 第一次提交会看到文件被改写，然后自作主张再次 add，可能死循环或提交格式化噪音。对策：skill 里写明"hook 修改文件后直接重试一次提交，不要改动这些文件"。
- **diff 太长会失真**。大改动时 Agent 只读到截断的 diff，总结会编造。要求它先跑 `git diff --stat`，超过 200 行就分块阅读关键文件。
- **密钥拦截**。我加了一条规则：暂存前检查 diff 中是否出现长随机串、`PRIVATE KEY`、`.env` 路径，命中即停。不能完全依赖，但能拦住大部分。
- **嵌套仓库**。monorepo 里 Agent 会在错误的根目录执行命令。skill 里强制先跑 `git rev-parse --show-toplevel` 校验工作目录。

## 可复用建议

- 工具面越小越好。白名单封装比"给 shell + 提示词约束"可靠得多——提示词是软约束，工具面才是硬约束。
- 仓库规范写成 skill 的一部分，而不是每次会话开头塞 prompt。前者可版本化、可复用到其他仓库。
- 保留人工确认点：push 和 main 分支操作。自动化收益最大的恰恰是本地动作，远端操作让给人。
- 用一个专用测试仓库先跑一周，看行为日志再上正式仓库。

## 总结

Git 自动化不是让 Agent 替你做决定，而是替你执行决定。真正要设计的是工具边界和确认策略，而不是把提示词写得花哨。这套东西一个下午就能搭起来，性价比在我目前所有 Agent 自动化实践里排第一。后续打算把 PR 描述生成和 review 预检也接进来，有进展再来发帖。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/45eb3106eab839e2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/8b8020bb17343f2f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/5a7cb752ae85cf5d.png)

