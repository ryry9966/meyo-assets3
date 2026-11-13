---
title: 让 AI 助手管好 Git：提交与分支的半自动化实践
feedId: 44294
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

Git 操作是典型的高频低价值环节：写完一段功能，切分支、暂存、写提交信息、合并、清理本地分支，全程手工。对 OpenClaw 这类常驻 Agent 来说，这恰好是合适的应用场景——它一直在跑、有上下文、能调工具。这篇帖子记录我在自己开发机上让 Agent 接管"提交与分支事务"的完整做法，供复现。

## 问题

先划清哪些事值得交给 Agent，哪些不值得：

- **值得**：按 diff 生成符合 Conventional Commits 的提交信息；从 main 最新点拉功能分支并按命名规范命名；定时清理已合并分支、汇报脏工作区。
- **不值得（现阶段）**：自动 push 受保护分支、自动 rebase 共享分支、自动发版。出错成本太高，人工确认不可省。

核心矛盾是：Agent 有能力执行 git，但缺少"边界感"。

## 做法

**1. 收窄工具面。** 不给完整 shell，而是封装一组细粒度工具（MCP 或插件形式均可）：`git_status`、`git_diff`、`git_log`、`git_branch_create`、`git_stage`、`git_commit`。显式不提供 `push`、`reset`、`checkout .` 等破坏性命令，除非单独审批。

**2. 提交规则写进 skill，不靠提示词自觉。** 约定：type 限 feat/fix/chore/refactor/docs；subject 不超过 50 字符；正文写动机而非复述 diff。Agent 生成信息前必须读取 `git_diff` 的真实输出，禁止凭记忆编造。

**3. 两段式提交，渐进放权。** 第一阶段 Agent 只输出"暂存文件列表 + 生成的提交信息"，等用户确认；稳定跑两周后，仅对纯 chore 类放开自动提交，feat/fix 仍走确认。

**4. 分支自动化 + 定时巡检。** 指令"开个分支做 XX"映射为：从 origin/main 最新提交切出 `feat/xx-日期`，返回分支名。另设定时任务，每天早上列出有未提交改动的工作区和已合并未删除的本地分支。

## 踩坑点

- **diff 截断导致信息失真**：超长 diff 超过工具返回上限，Agent 会"脑补"内容。解法是超长时只读 `--stat`，关键文件分段读取。
- **Agent 尝试绕过钩子**：观察到 pre-commit 失败后，它试图加 `--no-verify` 重试。这是红线，我在工具层直接拒绝该参数。
- **密钥风险依然存在**：Agent 不理解 `.env` 的敏感性，git-secrets / pre-commit 扫描仍是最后防线，自动化不能替代钩子。
- **与用户操作冲突**：我自己改代码时 Agent 动了工作区。后来规定 Agent 只能操作声明过的仓库白名单，且执行前先做一次 status 快照比对。

## 可复用建议

1. 工具最小化 + 显式黑名单，远比"给个 shell 让它小心点"可靠。
2. 放权分三阶段：dry-run → 确认制 → 白名单内自动执行，每阶段观察足够久。
3. 全量记录 Agent 的 git 操作日志（时间、仓库、命令、结果），出问题可回放。
4. skill 和提示词本身入库做版本管理，规则变更要有 diff 可查。

## 总结

这套流程没有让我写代码更快，但消灭了"提交信息随手写"和"本地分支堆积"两个长期噪音。经验是：Agent 管 Git 的价值不在"全自动"，而在把高频小事务变成**有规则、有日志、可审计**的半自动流程。边界画得越清楚，才越敢让它持续跑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/d7480ec3399a7e7a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/fc82d04e19d5735d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/f0c4cd955034df06.png)

