---
title: Git 自动化实战：让 Agent 管提交与分支，但别让它碰不可逆操作
feedId: 43864
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

写 commit message、清理僵尸分支、拆小提交——这些活不难，但琐碎且高频。团队接入 OpenClaw 之后，很自然想到把 Git 操作交给 Agent。跑了几个月，把经验沉淀成这篇帖子：怎么做才能既省事，又不把仓库搞坏。

## 问题

直接丢一句“帮我提交代码”给 Agent，通常会遇到三类问题：

1. **权限失控**：Agent 一旦能执行任意 git 命令，`push --force`、`reset --hard` 都可能被它“好心”执行。
2. **信息失真**：模型基于 diff 写提交信息时，容易脑补出 diff 里根本不存在的改动。
3. **边界模糊**：暂存区该不该动？切分支前未提交的修改怎么办？Agent 的默认行为和你的预期未必一致。

## 做法

我们的方案分三层：

**1. 工具层收敛。** 通过 MCP 暴露的 Git 工具只保留白名单：`status`、`diff`、`log`、`add`、`commit`、`branch`、`checkout`。`push` 单独封装，强制带确认参数；`push --force`、`reset --hard`、`clean` 干脆不暴露。

**2. 提交流程固化。** 在系统提示里写死流程：

```
1. git status + git diff --staged 确认待提交内容
2. 扫描 diff 是否含密钥特征（sk-、AKIA、PRIVATE KEY 等）
3. 生成符合 Conventional Commits 的信息，type/scope 必须能从 diff 推断
4. 输出拟执行的完整命令，用户确认后才 commit
```

**3. 分支策略脚本化。** 写了一个分支巡检 skill：每天跑一次，列出已合入 main 的本地分支和超过 14 天未活动的分支，输出清理建议清单。删除动作永远由人执行。

## 踩坑点

- **暂存区污染**：早期 Agent 会擅自 `git add .`，把本地调试文件带进提交。改成只 add 用户明确提到的文件后才稳定。
- **信息脑补**：有一次 commit message 写了“重构错误处理”，实际 diff 只是改了个变量名。解法是要求 Agent 附上 diff 变更行数摘要，超过阈值提示“信息可能夸大”。
- **上下文漂移**：长会话里 Agent 会忘记自己切过分支，继续在错误分支上提交。现在每次 commit 前强制重读一次 `git branch --show-current`。
- **密钥漏网**：关键词正则只能兜底，真正有效的是把 gitleaks 挂在 pre-commit 钩子里，Agent 的提交同样要过扫描。

## 可复用建议

1. **宁可慢，不要自动 push**。push 是最容易出事的不可逆操作，保留人工确认这一步。
2. **所有 Agent 的 git 操作落日志**。我们记了一份 JSONL，出问题能回放现场，排查效率高很多。
3. **提示词跟着仓库走**。把提交流程提示放进仓库级配置而不是全局设置，不同项目可以有不同规范。
4. **先只读，再写**。让 Agent 先跑两周“只分析不动手”的模式，观察它的判断质量，再逐步放开写权限。

## 总结

Git 自动化本身不难，难的是给 Agent 划清边界。核心原则一句话：**读操作放开，写操作收敛，不可逆操作必须过人**。这套流程跑下来，日常提交的规范度明显提升，分支堆积也少了——代价只是每次多一次确认点击，值得。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/cb969b2557a70e69.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/f9f0ad6b0864521c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/4e5af143990bdc63.png)

