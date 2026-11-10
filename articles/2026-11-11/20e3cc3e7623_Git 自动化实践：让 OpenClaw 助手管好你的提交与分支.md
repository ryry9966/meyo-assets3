---
title: Git 自动化实践：让 OpenClaw 助手管好你的提交与分支
feedId: 44064
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景

日常开发里，Git 操作占了不小的比重，但大多是低价值的重复劳动：写 commit message、整理分支名、删已合并的旧分支、cherry-pick 补丁。我们把这类活交给跑在本地的 OpenClaw 助手，让它通过消息频道直接操作仓库，跑了几周，把经验和坑整理如下。

## 问题：直接让 Agent 碰 Git 会翻车

最早的做法很朴素——给 Agent 一个完整 shell，让它自己 `git` 一把梭。结果三天出了四次事故：

- commit message 和实际 diff 对不上，典型的"看标题猜内容"；
- 一句 `git add .` 把 `.env` 和构建产物提进了暂存区；
- 在共享分支上尝试 rebase，差点触发 force push。

结论：不能给 Agent 无差别的 Git 权限，要在工具面和流程两个层面收口。

## 做法：三步搭一个克制的自动化

**1. 收窄工具面。** 不给裸 shell，用 MCP Git server 或自封装脚本只暴露白名单命令：`status`、`diff`、`log`、基于 `-p` 的分块暂存、`commit`、本地 `branch -d`。`push` 和 `rebase` 走独立入口，必须回到消息频道由人确认。

**2. 用 skill 固定工作流。** 在 OpenClaw 的 skill 文件里写死流程：先读 `git status` 和 `git diff`，禁止凭记忆描述变更；message 按约定式提交，正文写 why 而不是复述 what；任何破坏性操作必须先复述计划、等确认。skill 文件进版本库，团队共用同一套行为。

**3. 每周分支清理做成定时任务。** 定时触发：列出已合并且超过 14 天的本地分支，生成清单发到频道，人工点头后批量删除。远端分支默认只列不动。

## 踩坑点

1. **diff 超长撑爆上下文。** 先用 `--stat` 看范围，再按文件分片读，别一次塞全量 diff。
2. **Agent 迷恋 `git add .`。** 在封装层直接禁用，强制走显式路径或交互式分块。
3. **pre-commit 缺位。** 没配 hook 时 Agent 会提交不过 lint 的代码；后来把 hook 输出回传给 Agent，让它自己修完再提。
4. **"已提交"是幻觉。** Agent 偶尔声称完成但实际失败，要求它回读 `git log -1` 和 `status` 作为回执，问题基本消失。
5. **清理分支误伤 CI。** 有次删掉了还在跑流水线引用的分支，现在清理前先查 CI 状态。

## 可复用建议

- 所有破坏性命令（force push、`reset --hard`、`clean`）走独立的确认通道，不进常规工具面；
- 让 Agent 每次输出"操作前/操作后"两段状态快照，方便事后审计；
- 从单人仓库灰度开始，稳定两三周再扩展到多人协作仓库；
- 规则写在 skill 文件里而不是聊天记忆里，否则换台机器行为就漂移。

## 总结

Agent 管 Git 的价值不在全自动，而在分层：高频低风险的动作（message、本地清理）自动跑，低频高风险的动作（push、rebase）降级成需要人点头的流程。OpenClaw 在这件事里的正确定位是一个守规矩的操作员，而不是自由发挥的开发者——权限给得越少，它反而越可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/852cbf35465a3c75.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/449c576ccdcd9a76.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/e3f067406a17fad4.png)

