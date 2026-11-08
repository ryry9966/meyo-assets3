---
title: Git 自动化实战：让 AI 助手管提交和分支，先把权限关进笼子
feedId: 43787
source: 综合讨论
publishedAt: 2026-11-08
---

## 背景

OpenClaw 接上 MCP 的 git server（或给 Agent 一个受限 shell 工具）之后，最适合拿来落地的第一件事就是 Git 自动化。原因很朴素：写 commit message、清理本地分支、按 issue 开分支，这些活重复、机械、判断规则明确，正好是 Agent 擅长的部分。

但社区里翻车案例也不少：Agent 把没 review 的改动一股脑 commit、自动 push 到了主分支、甚至执行了 `reset --hard`。这里分享一套我们在自己仓库跑了几个月的做法——让 Agent 干活，但把危险操作关进笼子。

## 问题

手动 Git 工作流有三个痛点：

1. commit message 质量不稳定，事后翻 log 基本靠猜；
2. 本地分支越攒越多，命名混乱，谁也不敢删；
3. 每次人工 review diff 太繁琐，经常顺手就 `git commit -am "fix"`。

反过来，把 Git 完全交给 Agent 风险也很明确：它会替你做决定，而 Git 的很多决定（push、rebase、reset）是不可逆的。

## 做法

方案分三层：权限收敛、流程约束、规则注入。

**第一层：权限收敛。** 不给裸 shell，而是通过 MCP git server 或命令白名单，只放行：

```
git status / diff / log / branch / add / commit / stash / switch -c
```

明确拒绝：`push --force`、`reset --hard`、`clean -f`、`rebase`、`cherry-pick`。普通 push 可放行，但要求执行前 Agent 必须输出计划并等人工确认——在 OpenClaw 里可以用"敏感操作需确认"的插件规则实现。

**第二层：流程约束。** 写进 Agent 系统提示，作为硬性步骤：

1. commit 前必须先跑 `git status` 和 `git diff --staged`，确认暂存内容与本次任务相关，发现无关文件先停下来问；
2. message 遵循 Conventional Commits，格式 `type(scope): 摘要`，摘要从 diff 提炼，不许编造没发生的改动；
3. 新分支一律从最新 main 拉，命名 `type/issue号-短描述`；
4. 多文件改动拆成原子 commit，不一锅炖。

**第三层：规则注入。** 仓库根目录放一份约定文件（分支命名、commit 类型、禁改目录），提示里指明路径，让 Agent 开工前先读。

跑下来之后，我的日常变成：改完代码说一句"按约定提交"，Agent 给出 diff 摘要和拟好的 message，我扫一眼确认，完事。分支清理也从"季度大扫除"变成 Agent 每周列出已合并分支、我点头删除。

## 踩坑点

- **暂存区污染**：最常见的错误是把上次任务残留的暂存文件一起 commit。解法是强制"commit 前必看 status"，确认环节展示文件清单。
- **长 diff 截断**：大改动时 Agent 看不全 diff，message 会泛化成"update files"。几百行以上的 diff，让它按文件分批读，或基于任务描述+文件名来写。
- **交互式命令卡死**：`git rebase -i`、merge 冲突提示都是交互式的，Agent 会挂住。只允许非交互参数，冲突一律停下交人工。
- **密钥入库**：Agent 对 `.env`、私钥没有天然警觉。禁改清单写敏感路径，另加 pre-commit 钩子兜底扫描。
- **detached HEAD**：Agent 在 checkout 旧 commit 后继续提交，改动会丢。规则明确：只允许 `switch -c` 开新分支，禁止裸 checkout commit hash。

## 可复用建议

1. 默认只读，写操作逐条放行——Git 自动化的第一原则；
2. 让 Agent 先输出计划（commit 哪些文件、message 是什么）再执行，计划即确认材料；
3. 团队约定别散落在聊天里，落成仓库文件，Agent 和人共用同一份真相；
4. pre-commit 钩子做最后一道防线，机器写的 message 同样要过 lint 和密钥扫描。

## 总结

Git 自动化不是让 Agent 替你思考，而是让它处理机械部分、你保留决定权。权限收敛 + 流程约束 + 人工确认这三件事搭好之后，AI 助手管理提交和分支是稳定可用的，而且比想象中省时间。建议从"只帮忙写 commit message"这个最小场景起步，跑稳了再逐步放权。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/92147111d02f36cd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/15c7fbfe1846eb60.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/f32b701cdd6f3608.png)

