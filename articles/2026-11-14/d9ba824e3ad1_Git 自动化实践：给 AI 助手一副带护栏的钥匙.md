---
title: Git 自动化实践：给 AI 助手一副带护栏的钥匙
feedId: 44354
source: 综合讨论
publishedAt: 2026-11-14
---

## 背景

日常开发里有一类工作高度重复但并不难：写提交信息、切分支、暂存切换、整理变更。这类事单次不占多少时间，但频繁打断心流。OpenClaw 这类常驻助手的价值恰恰在于承接这种"小而高频"的操作——前提是权限设计要先行。

## 问题

最常见的错误起点，是直接把裸 shell 交给 agent。风险集中在三点：

- **破坏性命令**：`reset --hard`、`push --force`、`clean -fd`，一次误操作就是半天工作量
- **状态幻觉**：agent 拿着几轮对话前的旧 `git status` 就动手，提交内容和预期不符
- **提交质量差**：不加约束的 agent 会输出 "update" 这种提交信息，或者一把 `git add -A` 把 `.env` 和构建产物一起提进去

## 做法

### 1. 先收敛执行面

不要给裸 shell。写一个 `safe-git` 包装脚本，只放行白名单子命令：

```bash
ALLOW="status diff log add commit branch switch stash restore show"
```

`push`、`reset`、`rebase`、`clean` 一律拦截，人工执行。每次调用顺手写入日志文件，事后可审计。把这个脚本作为 MCP 工具或插件暴露给 agent，替代通用 shell 工具。

### 2. 提交流程约束

在仓库级的 agent 指令文件里固化规则：

- 提交前必须现场执行 `git status` 和 `git diff --stat`，禁止复用对话中的旧状态
- 先看 stat 分组，再按文件看 diff，把变更拆成逻辑提交，遵循 Conventional Commits
- `git add` 前先列出待加文件逐个确认，不用 `-A`

### 3. 分支管理

- 命名规范写进指令：`feat/`、`fix/`、`chore/` 前缀加短横线描述
- 建分支前先 `git branch --list` 查重，避免同名残留分支
- 一条铁律：agent 永远不碰 main，一切工作在特性分支上，push 始终由人工触发

我的日常用法是一个"收工提交"指令：下班前让 agent 汇总当日变更、起草提交拆分方案、列出待确认文件，我确认后它才执行 commit。

## 踩坑点

- **交互式命令会挂起**：`git rebase -i`、`git add -i` 在工具调用里会卡死等待输入，一律换非交互等价方式
- **大 diff 导致幻觉**：变更超过几百行时 agent 容易开始编造，务必先 `--stat` 再分片看
- **换行符噪音**：CRLF 混乱的仓库里，agent 会把行尾变更当成真实改动提交。接手自动化前先统一 `.gitattributes`
- **凭据不进对话**：token 走 SSH agent 或凭据管理器，任何情况下不要让凭据出现在提示词里

## 可复用建议

- **渐进放权**：第一周只读（agent 只建议不执行），第二周放开 commit，push 保持人工最久
- 包装脚本独立成文件，跨项目复用，别散落在各个 prompt 里
- 约范（分支命名、提交格式）放仓库级指令文件，而不是每次在对话里重复
- 日志是底线：agent 跑过的每条 git 命令都该能回溯

## 总结

让 AI 管 Git，核心不是自动化有多炫，而是约束设计有多窄。白名单、分支隔离、人工 push 这三道护栏立住之后，agent 带来的实际收益是提交质量和上下文切换成本的改善，而不是"全自动"。把它当成一个守规矩的初级同事来授权，比当成自动驾驶来期待，要靠谱得多。欢迎在评论区交流你们的护栏做法。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/f5faf54c3921c05d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/2dd703cf3fa0c81a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/e8e0901cf9cf9e3c.png)

