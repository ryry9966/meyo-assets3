---
title: 让 AI 助手接管 Git 日常：提交与分支自动化的工程实践
feedId: 43672
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

我们组在 OpenClaw 上做插件开发，每人手上同时挂着四五个仓库。真正写代码的时间其实不多，更多时间花在：写提交信息、切分支、清理已合并分支、push 前整理变更说明。这类工作认知负担低但重复度高，很适合交给常驻的 agent。

## 问题

直接丢一句"帮我管理 git"会出两类事：

1. **质量问题**：生成的 commit message 千篇一律，甚至和实际 diff 对不上；
2. **安全问题**：agent 拿到完整 git 权限后，可能误删分支、误 push、碰了 main。

所以目标不是全自动，而是：**读操作全自动化，写操作半自动化**——agent 准备，人确认。

## 做法

### 1. 收敛工具面

通过 MCP git server 只暴露最小工具集：`status / diff / log / branch_list / commit / create_branch`。`push / reset / rebase / --force` 一律不开放，需要 push 时我自己在终端敲。

### 2. 规范进仓库，不进 prompt

写一份 `GIT_RULES.md` 放在仓库根目录，agent 每次任务开头先读：

```markdown
- commit message 遵循 conventional commits，英文，首行 ≤ 72 字符
- 禁止操作 main / release/* 分支
- 提交前必须输出 git status 和待提交文件清单
- 写操作先给出计划（dry-run），人工确认后执行
```

规范随仓库走，团队成员的 agent 行为一致，不用各自维护 prompt。

### 3. 三个高频场景

- **提交助手**：agent 读完 diff，按逻辑拆成若干次提交，每次给出 message 和文件清单，我逐条确认。比一次性大提交干净得多。
- **分支清理**：每周五定时任务，agent 列出所有已合并进默认分支的本地分支并输出报告，我回复"删 3、5"它才执行。
- **push 前摘要**：自动生成自上次 push 以来的变更草稿，直接贴进 MR 描述。

### 4. 留痕

所有 agent 的 git 写操作追加到 `~/.openclaw/git-audit.log`，带时间戳和仓库路径，出问题能回溯。

## 踩坑点

- **大 diff 截断**：单文件 diff 超几百行后，agent 会"脑补"没看到的上下文，message 开始编造。解法是按文件分批读，超长文件只给函数级摘要。
- **pre-commit hook 静默改文件**：hook 格式化后，agent 记录的文件清单和实际提交不一致。现在提交后强制再跑一次 `status` 校验。
- **定时任务不幂等**：分支清理跑两次会重复报错。改成"先查状态，状态没变才动手"的模式。
- **中英文规范打架**：初期生成的中文 message 与团队英文规范冲突，在 GIT_RULES 里写死后才稳定。

## 可复用建议

1. 权限分层是前提：只读工具默认给，写工具白名单 + 人工确认；
2. 规范写成仓库内文件，别散落在个人 prompt 里；
3. 先在自己的玩具仓库跑两周，观察失败模式，再进团队仓库；
4. 写操作必须有审计日志——排障时这是唯一证据。

## 总结

这套流程跑了一个多月，提交粒度和 message 质量明显变好，周五清分支从十分钟变成点一下确认。核心经验一句话：**让 AI 做"读和准备"，人做"写和确认"**。全自动听起来性感，但 git 恰是不可逆操作的高发区，半自动才是能长期跑下去的形态。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/067564d5dcfacf0c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/bf5bd12947813916.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/9d00feb9abbf4328.png)

