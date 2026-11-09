---
title: Git 自动化实践：让 OpenClaw 助手管理提交与分支，以及如何管住它的手
feedId: 43945
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景

OpenClaw 助手具备 shell 执行和技能扩展能力后，最自然的落地场景之一就是 Git。日常开发里有大量机械操作：写提交信息、切分支、清理已合并分支、临时 cherry-pick。这些操作规则清晰、可枚举，适合交给助手；但 Git 直接作用于代码仓库，权限给大了就是事故。这篇帖子分享我们团队跑了两个多月的方案，重点讲如何划边界。

## 问题

实际痛点有三个：

1. 提交信息质量不稳定，review 时经常要追问“这个 commit 到底改了啥”；
2. 分支腐化，本地堆着几十个已合并分支，谁也不敢乱删；
3. 上下文切换成本——正写代码时被打断去处理紧急操作，回来就丢了状态。

## 做法

### 1. 用最小权限封装 Git 技能

不要把裸 shell 直接交给助手。我们写了一个技能（skill），内部维护命令白名单：`status`、`diff`、`log`、`branch`、`add`、`commit`、`switch` 默认放行；`push`、`reset`、`rebase`、`clean` 需人工确认；`push --force`、`--no-verify` 直接拒绝。关键是这层约束做在脚本里而不是 prompt 里——prompt 是软约束，脚本是硬约束。

### 2. 提交信息生成：diff 分层喂给模型

把整个 `git diff` 塞进上下文，大 diff 会爆 token 且信息被稀释。流程是：先 `git diff --stat` 拿文件级概览，再按需取关键文件的完整 diff（截断到固定行数），生成符合 Conventional Commits 的消息，打印 preview 等确认后再执行 `git commit`。

### 3. 分支治理：先报告，后清理

配一个定时任务，每周生成分支报告：哪些分支已合并进 main、哪些超过 30 天无提交、哪些远程分支已删除。第一版只报告不删除，跑了一个月确认误报率可接受，才加了 `git branch -d`（注意是 `-d` 不是 `-D`）的确认式清理。

## 踩坑点

- **staged 与 unstaged 混淆**：助手一度基于工作区 diff 写提交信息，但实际 staged 内容不同。修复：生成前强制先跑 `git status --porcelain` 核对，不一致就提醒用户。
- **pre-commit hook 改文件**：hook 格式化代码导致暂存内容过期。解法是 commit 失败后重新走一遍 status → diff → commit 流程，而不是盲目重试同一条命令。
- **多行提交信息转义**：经 shell 传递换行符容易出问题，统一改用 `git commit -F msgfile` 写临时文件，比 `-m` 拼接可靠得多。
- **误上错分支**：新增规则——写操作前必须回报当前分支和上游状态，分支名不在预期列表就中止。

## 可复用建议

1. 所有“确认式”操作先打印完整命令再执行，并把助手执行过的每条 git 命令落盘成日志，方便回溯。
2. 危险操作封装成带二次确认的脚本，技能只暴露脚本入口。
3. 自动化优先做“读”（报告、诊断），写操作永远留人工闸门。
4. 把分支命名规范、commit 规范写进技能文档，助手会照着自检。

## 总结

两个多月下来，这套方案没有出过一次需要回滚的事故——原因不是模型多聪明，而是权限窄、有日志、写操作有人工闸门。Git 自动化的价值不在“全自动”，而在于把规则明确的那部分机械劳动交出去，把判断留给人。建议有类似需求的同学从分支报告这类只读场景起步，跑稳了再逐步放开写权限。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/b2db791545d2635e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/83a60f627ca0573f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/a38f4c8005710f7e.png)

