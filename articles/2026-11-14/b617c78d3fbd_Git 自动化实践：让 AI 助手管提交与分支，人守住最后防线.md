---
title: Git 自动化实践：让 AI 助手管提交与分支，人守住最后防线
feedId: 44396
source: 综合讨论
publishedAt: 2026-11-14
---

## 背景

最近几个月我的日常开发基本变成了"提需求 + review"。Agent 直接在仓库里改代码，效率确实上来了，但很快暴露出另一个问题：agent 干完活就走人，工作区留下一堆未跟踪文件，commit message 全是 `update`，偶尔还在 main 上直接提交。AI 写代码的速度上来了，git 管理没跟上，反而成了新瓶颈。

## 问题

让 agent 自由使用 git，典型事故有三类：

1. **脏提交**：`git add -A` 把 `.env`、日志、构建产物一并提交；
2. **无效历史**：commit message 缺乏上下文，回溯时等于没有历史；
3. **危险操作**：在错误分支上操作、`reset --hard`、甚至 `push --force`。

## 做法

我的方案是三层结构：**受限工具 + 明确 SOP + hook 兜底**。

**第一步：收敛工具面。** 不给 agent 裸 shell，而是通过 MCP 的 git server（如 mcp-server-git），或自封装一个只暴露白名单子命令（`status / diff / add / commit / branch / checkout / log`）的脚本。`push`、`reset`、`rebase` 一律不在白名单内。

**第二步：把 SOP 写进 agent 指令。** 核心规则：

- 所有工作在 `agent/<任务名>` 分支上进行，禁止直接在 main 提交；
- 提交前必须先输出 `status` 和 `diff --stat`，并给出变更摘要，人工确认后才执行 commit；
- message 遵循 Conventional Commits，正文写清"为什么"，不只是"改了什么"；
- 大改动按逻辑拆成多个小提交。

**第三步：hook 兜底。** pre-commit 挂 gitleaks 扫敏感信息，lint-staged 保证格式。这一层是给 agent 犯错时兜底的，不能省。

## 踩坑点

- **分支错位**：agent 曾在 detached HEAD 上提交，HEAD 一动 commit 就成了孤儿对象。后来在 SOP 里强制"提交前执行 `git branch --show-current` 并自报分支"才堵住。
- **并行任务互踩**：两个 agent 共用一个 worktree 时会互相污染暂存区。用 `git worktree` 给每个任务独立目录后就干净了。
- **提交太"勤快"**：早期允许自动 commit，历史里全是碎提交。改成"摘要 + 人工确认"后历史质量明显回升——牺牲一点自动化换可读性，值。

## 可复用建议

1. SOP 写进仓库的 `AGENTS.md`，随代码分发，团队所有 agent 共用一份规则；
2. 白名单封装优于权限放养，宁可 agent 多调几次工具，也不要让它有破坏能力；
3. push 权限永远留在人手里，agent 产出止步于本地 commit + 分支，合入走 PR；
4. 每周跑个脚本清理残留的 `agent/*` 分支，避免分支沼泽。

## 总结

AI 管 git 的正确姿势，不是替你做决定，而是把机械步骤（暂存、建分支、格式化 message）交出去，人保留 review 与合入的最终防线。这套三层结构在两个真实项目里跑了一个多月，脏提交归零，历史可读性显著提升。迁移成本很低，建议从"commit 前强制输出变更摘要"这一条规则开始试。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/c3f5dd224acff039.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/7ebf25c15acc2840.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/96d9be6c8e08dc43.png)

