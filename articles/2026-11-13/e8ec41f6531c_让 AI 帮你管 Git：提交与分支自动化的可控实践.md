---
title: 让 AI 帮你管 Git：提交与分支自动化的可控实践
feedId: 44345
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

日常开发里，Git 操作占了大量碎片时间：拆分提交、写符合规范的 commit message、清理过期分支、按任务建分支。OpenClaw 这类常驻 Agent 天然带工具执行能力，把它接进 Git 工作流是顺理成章的事。但"让 AI 直接操作仓库"和"让 AI 帮我操作仓库"是两回事，这篇记录我们落地三个月后的实际做法。

## 问题

最初的目标很朴素：

- commit message 质量差，一堆 `update`、`fix bug`，回溯时基本没用；
- 一次改动常混着格式化、重构和逻辑变更，AI 一把梭提交后没法单独 revert；
- 分支堆积，release 后没人记得删；
- 最担心的还是安全：误操作 force push、把临时文件和密钥提交进仓库。

## 做法

原则是"AI 起草，人来确认"，分五步：

1. **先给约定，再给权限。** 在仓库根的 AGENTS.md 里写清楚：Conventional Commits 格式、分支命名 `feat/xxx`、`fix/xxx`、禁止 `--force`、禁止未确认直接 push。Agent 每次会话都能读到，比在聊天里反复交代可靠得多。
2. **限制输入源。** 要求 Agent 先跑 `git status` 和 `git diff --staged`，基于真实 diff 写 message，而不是按文件名猜。这是 message 质量的分水岭。
3. **两段式提交。** 固定成一条自定义指令（类似 `/commit` skill）：staged diff → AI 产出 3 个候选 message → 人选一个或改 → 执行 commit。全程不碰未 staged 的内容。
4. **用 MCP git server 收窄工具面。** 相比直接开放 shell，MCP 的 git 工具只暴露 status / diff / log / commit 这类细粒度操作，误执行的面小很多。需要 rebase、cherry-pick 时再临时开 shell 并要求 dry-run。
5. **分支自动化的边界。** 允许：按任务建分支、列出超过 30 天未动的分支并生成清理建议、release 后整理 merge 记录。不允许：删除他人分支、任何写历史的操作——这些只生成命令，人来执行。

## 踩坑点

- **`git add -A` 是重灾区。** Agent 图省事会一把 add，把构建产物、`.env` 提交进来。后来在规则里明确"只 add 明确列出的文件"，pre-commit 里加密钥扫描兜底。
- **message 语言不一致。** 仓库历史是英文 message，AI 默认写中文，把"commit message 用英文"写进规则才稳定。
- **AI 会幻觉 scope。** `feat(auth)` 里的 auth 可能根本不存在，需要在规则里给出模块清单。
- **Windows 的 CRLF。** Agent 执行 `git config core.autocrlf` 想"帮忙"，反而搞乱团队配置，全局配置列为禁改项。
- **rebase 冲突别让 AI 自动解。** 它倾向"能过就过"，冲突的语义判断还是得人来。

## 可复用建议

- Git 约定写进仓库级规则文件，别靠对话记忆；
- 危险操作走"生成命令 → 人执行"，AI 只碰只读 + 低风险写操作；
- 用 allowlist/denylist 包一层命令过滤，denylist 至少包含 `push --force`、`reset --hard`、`filter-branch`；
- 把 `/commit` 这类固定流程沉淀成 skill，团队共用同一套，输出才可预期。

## 总结

这套流程跑下来，commit message 可读性明显改善，过期分支从二十几个降到个位数。但真正的收益不是省时间，而是把 Git 操作从"凭手感"变成"有约定、有边界、可审计"。AI 助手在这里的角色是起草者和整理者，最终按回车的那个人，还是你自己。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/e83724b0fb2fc544.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/94571acb90d41ead.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/63f1e763b3781941.png)

