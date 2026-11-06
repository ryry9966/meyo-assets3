---
title: 让 Agent 帮你管 Git：提交与分支自动化的落地实践
feedId: 43656
source: 综合讨论
publishedAt: 2026-11-06
---

## 背景

团队规范一直有：Conventional Commits、分支按 `类型/issue-短描述` 命名、合并后及时清理。规范不难，难的是每次都执行。我每天的提交里大概七成 message 是临时编的，分支名随手起，`main` 附近常年挂着十几个已合并没删的分支。

OpenClaw 的 agent 有工具执行能力（shell、MCP 工具、定时任务），这件事理论上可以直接交出去。我花了两周做了一版"让 agent 管 Git"的流程。先说结论：可行，但前提是把边界画死。

## 问题

最初的做法很天真：给 agent 开 shell，让它直接跑 git。三天出了两次事故——

- 一次 `git add -A` 把本地 `.env` 带进了暂存区，差一步就 push；
- 一次工作区有未提交改动时 agent 直接 switch 分支，冲突糊了一屏，我手工收拾了半小时。

根因不是模型不行，是我没给它围栏。

## 做法

**1. 白名单脚本替代裸 shell。** 写了个 `git-wrapper`，只放行 `status / diff / add / commit / branch / switch / log / push`，`--force`、`reset --hard`、`clean`、`rebase` 一律拒绝并提示走人工。agent 无论走 shell 还是 MCP 工具，唯一入口都是这个脚本。

**2. 提交流程固化成三步。** agent 先跑 `git status` 和 `git diff --staged`，再按模板生成 message（类型前缀 + issue 号 + 正文说明"为什么改"），正文必须引用 diff 里实际存在的片段；commit 初期由我执行，跑顺后改为 agent 执行、我事后抽查。

**3. 分支自动化。** 从 issue 标题生成分支名；建分支前强制 `git status`，工作区脏就先 stash 并报告，而不是硬切。

**4. 定时巡检。** 用 OpenClaw 定时任务每天跑一次：列出已合并进 main 的分支和超过两周未动的分支，只输出报告，不执行删除。

**5. 审计日志。** wrapper 把每条命令和退出码追加到本地日志，出问题能回放 agent 到底干了什么。

## 踩坑点

- **`git add -A` 是重灾区。** 改成 agent 先列文件清单、排除 `.env`、密钥、构建产物类路径，再逐个 add。
- **message 幻觉。** 模型会描述 diff 里不存在的"改进"，对策是要求正文可对照 diff，抽查不过就打回重写。
- **长 diff 撑爆上下文。** 先 `--stat` 看全貌，再挑文件看细节，不要整段 diff 直接塞。
- **提交身份错乱。** 容器里 `user.name/email` 没配，提交记录一团糟，用 per-repo config 固定掉。
- **共享分支的历史改写必须人做。** rebase、force-push 直接进黑名单，没有商量余地。

## 可复用建议

- 一条原则：**只读和增量操作可以放开，不可逆操作必须人在环。**
- 白名单和 message 模板放进仓库（比如 `.agent/` 目录），团队共用一份，别靠口头约定。
- 先跑一两周"报告模式"（agent 只起草 message 和分支名，不动手），确认质量再逐步放权。
- 用了 Conventional Commits 之后，changelog 和版本号整理也可以顺势交给 agent，收益是连带的。

## 总结

AI 管 Git 的价值不在"全自动"，而在把高频、低风险、格式化的部分（message 起草、分支命名、巡检报告）自动化，把不可逆操作留给人。围栏、审计、灰度放权，这三件事做到位，agent 才是助力而不是事故源。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/aaba9610c7947b60.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/b61ffb4ff7e21ba4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/95cc50ea6cf4f3ec.png)

