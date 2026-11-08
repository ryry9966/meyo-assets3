---
title: 让 AI 助手接手 Git 体力活：提交信息与分支治理的半自动实践
feedId: 43843
source: 综合讨论
publishedAt: 2026-11-08
---

## 背景

OpenClaw 这类 agent 平台跑通之后，最先产生收益的往往不是"让 AI 写代码"，而是写代码周边的体力活。Git 日常操作里，写提交信息、按规范起分支名、清理合并后忘删的本地分支，占掉不少碎片时间。这些任务模式固定、判断量小，正好适合交给 agent 半自动化处理。

## 问题

直接把 shell 丢给 agent 并不现实，实际踩到三个痛点：

1. **权限风险**。agent 能执行任意命令，意味着一次幻觉触发的 `git reset --hard` 或误删分支就是真实损失。
2. **信息以偏概全**。staged diff 几百行时，模型生成的提交信息常常只覆盖它"看全"的那部分。
3. **规范靠自觉**。Conventional Commits 写在文档里没人看，agent 如果不强制读取，输出照样跑偏。

## 做法

**第一步：受限工具面，不给裸 shell。** 通过 MCP 接 git server，或自己封一层 CLI 插件，只暴露 `status`、`diff`、`log`、`add`、`commit`、`branch -d` 这类白名单子命令。`push`、`reset`、`clean` 一律不进工具列表，物理上杜绝危险操作。

**第二步：提交信息走"先读后写"。** 固定流程：agent 先跑 `git diff --staged --stat` 看全貌，再按需拉取关键文件 diff，然后用一句话复述变更意图，最后按 Conventional Commits 产出 message。要求它只输出命令不执行，人工过目后再跑。多花十秒，省掉一条 "fix bug" 式提交。

**第三步：分支治理做成周期任务。** 让 agent 每周跑一次：列出所有已合并到 main 的本地分支、生成清理清单、标注最后提交时间，确认后执行 `branch -d`（只允许 `-d`，不带 `-D`）。新建分支时，agent 负责按 `feat/issue-123-short-desc` 格式纠正命名。

**第四步：钩子兜底。** 在仓库 `commit-msg` hook 里做格式校验，agent 生成不规范也过不了关。规范本身写在仓库的 `AGENTS.md` 里，agent 每次任务都会读取。

## 踩坑点

- **index.lock 冲突**：agent 执行期间手动操作同一仓库，容易撞锁。约定单仓库同一时间只有一方操作。
- **幻觉引用**：模型会编造 issue 编号。现在明确要求：上下文里没有的编号不许写。
- **大 diff 截断**：超过一定行数就让 agent 先出 `--stat` 摘要并声明覆盖范围，而不是硬读全文。
- **中英混排**：中文 subject 在部分 CI 解析里出问题，统一英文 subject、说明放 body。

## 可复用建议

- 所有写操作先出计划、再执行，两步分离是安全底线。
- agent 执行过的每条 git 命令落日志，出问题能回溯。
- 整套流程沉淀成一个 skill 或插件在团队内共享，比口头约定可靠得多。

## 总结

AI 管 Git 的正确姿势不是"全自动"，而是"agent 干体力活、人守关键节点"。白名单工具面、先读后写、钩子兜底，三件事做齐，提交信息和分支治理基本可以放手。剩下真正需要判断的事——要不要合、要不要删——仍然留在人手里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/2b831278b6bac366.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/10f0e36f3dbec4a5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/ad3f6d3ca176bf32.png)

