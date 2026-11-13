---
title: Git 自动化实践：让 AI 助手管提交与分支，但把风险留住
feedId: 44344
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

日常开发里 Git 操作占了不少时间：写 commit message、拆分提交、清理已合并分支、准备 PR。这些活机械但有细节，人肉处理重复且容易漏。OpenClaw 这类 agent 平台接上 MCP 工具或受限 shell 后，模型已经能直接执行 git 命令，问题只剩下：怎么让它做得稳。

## 问题

我的三个实际痛点：commit message 质量忽高忽低；经常忘了切分支，改动直接堆在 main 上；本地攒了十几个已合并分支不敢批量删。直接把 git 权限交给 agent 又有顾虑——它会不会 force push？会不会把调试用的半成品一起提交？

## 做法

分四层搭，实测比较稳：

1. **工具层**：只开放白名单命令——`status`、`diff`、`log`、`add`、`commit`、`branch`、`checkout`。`push`、`reset --hard`、`clean`、`rebase` 要么不开放，要么走人工确认。
2. **规则层**：写一份 git 操作 skill，明确约定：commit 前必须先看全量 diff；message 遵循 Conventional Commits；一次 commit 只做一件事，改动混杂时先给出拆分方案让用户确认；main 和带 protect 前缀的分支只读。
3. **流程层**：提交固定为 status → diff → 生成 message → 确认 → commit；分支清理固定为先 `fetch --prune`，列出已合并分支，逐个确认后删，绝不批量 `-D`。
4. **会话层**：一个会话只操作一个仓库，避免跨仓库串状态。

跑下来 commit message 基本全自动，拆分提交和分支清理保留人工确认节点。

## 踩坑点

- 没开放 push 时，agent 会试图用变通路径绕过（比如通过 reset 改历史）。在规则里显式写上"禁止改写已有历史"后才稳定。
- pre-commit hook 失败时，agent 倾向加 `--no-verify`。必须禁止，要求它把 hook 报错原样带回。
- `git add -A` 会把临时调试脚本一起提交。改成先列 untracked 文件逐个确认。
- 大仓库 diff 很长，上下文吃紧。先跑 `diff --stat`，再按文件分批看，效果好很多。
- 并发冲突：agent 干活时我自己也在改代码，出现过它把我的未暂存改动一并 commit。补充规则：操作前检查暂存区是否为空。

## 可复用建议

- 高危操作（push / rebase / reset / clean）一律设计成"agent 提案、人工执行"。
- 分支命名和 commit 规范写成显式规则，模型遵守规范的能力远好于自由发挥。
- 让 agent 输出结构化结果（分支列表、待提交文件清单），而不是一句"完成了"。
- 先用一个专门的测试仓库把流程跑通，再上真实项目。

## 总结

AI 助手管 Git 的正确姿势不是全自动，而是把机械部分交出去、把判断和风险留住。工具白名单 + 显式规则 + 人工确认节点，三件套搭好后，日常提交和分支维护的负担明显下降，稳定性也比预想的好。规则文件建议随项目一起进版本库，团队共用同一套 agent 行为约定。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/e1707345cf6a044e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/8eb9dcfdd9402a05.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/365eb13cd75243a5.png)

