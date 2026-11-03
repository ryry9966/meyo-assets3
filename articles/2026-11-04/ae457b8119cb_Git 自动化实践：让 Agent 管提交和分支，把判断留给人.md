---
title: Git 自动化实践：让 Agent 管提交和分支，把判断留给人
feedId: 43380
source: 综合讨论
publishedAt: 2026-11-04
---

## 背景

用 Agent 写代码之后，Git 操作的频率明显上来了：一小段功能一个提交、试验性改动随手开分支，一天十几条 commit 很常见。这时真正耗时间的不再是写代码，而是补 commit message、整理分支、决定改动怎么拆——都是典型适合交给自动化的事。

## 问题

直接给 Agent shell 权限让它"随便提交"，实践中会遇到三类麻烦：

1. **提交信息失真**。模型容易写出"看起来合理"的 message，但和实际 diff 对不上，回溯时反而误导。
2. **危险操作**。`git add -A` 可能扫进密钥文件；`push --force`、对共享分支做 `reset --hard`，一次失误就是团队事故。
3. **分支堆积**。Agent 每个任务开一个分支，结束没人清理，两周后 remote 上全是 `agent/fix-xxx`。

## 做法

我的方案是给 Agent 配一层受限的 Git 工具集（MCP 的 git server，或者自己包一层脚本），核心是四条规则：

**1. 命令白名单。** 只暴露 `status / diff / log / add <path> / commit / branch / switch / merge --no-ff`。`push` 单独走人工确认；`reset --hard`、`push --force`、`rebase` 干脆不提供。

**2. 提交流程固定三步。** Agent 必须先跑 `git status` 和 `git diff --staged`，引用实际改动写 message（遵循 Conventional Commits），列出待提交文件等我确认，确认后才执行。不允许 `add .`，只允许显式路径。

**3. 分支有命名和生命周期。** 统一 `agent/<task-slug>` 前缀，任务合并后由 Agent 发起清理：列出已合并到 main 的本地分支，删除前再确认一次。我另留一个每周跑的脚本兜底。

**4. 写操作落日志。** 每次 git 写命令，把命令、时间、commit hash 追加到本地日志，出问题能定位到具体一步。

## 踩坑点

- **密钥险些入库**：有次 Agent 把 `.env.local` 一并 add 了，靠 pre-commit 挂的 gitleaks 才拦住。Agent 的自觉不可信，secret 扫描必须是硬性 hook。
- **message 与 diff 脱节**：早期允许 Agent 不看 diff 就写 message，结果把三个不相干改动合成了一条 `fix:`。把"引用 diff 原文"写成硬性要求后才稳定。
- **错误的工作区**：会话残留 detached HEAD，Agent 直接在上面提交。现在第一步永远是 `git status` 并确认当前分支。
- **amend 已推送提交**：Agent 曾建议 `commit --amend` 一个已 push 的提交，被白名单拦下。规则要在工具层挡，不能靠提示词自觉。

## 可复用建议

- 规则写在 skill/系统提示里不如写在工具封装层：前者会被"创造性"绕过，后者不会。
- 磨合期开 dry-run：所有写操作先打印将执行的命令和预期结果，人工放行。
- message 可以让模型写，但**拆分提交的决策留给人**——这是唯一需要业务判断的点。
- 定期让 Agent 生成分支健康报告（未合并、长期未动、已合并未删），比全自动清理稳妥。

## 总结

Git 自动化的收益不在"省下写 commit message 的三十秒"，而在于把提交纪律变成确定性流程。跑下来，Agent 承担了 90% 的机械操作，人只在两个点介入：确认提交内容、决定提交拆分。关键是权限收窄加硬性 hook，让错误在发生前被拦截，而不是事后回滚。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/26cf4c4c5e90a1e1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/a88f4598c6dd2ae4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/90b377fa6811b537.png)

