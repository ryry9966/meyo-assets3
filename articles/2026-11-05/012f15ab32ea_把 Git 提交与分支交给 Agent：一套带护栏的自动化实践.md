---
title: 把 Git 提交与分支交给 Agent：一套带护栏的自动化实践
feedId: 43477
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

日常开发里最消耗心力的往往不是写代码，而是提交纪律：`git status` 里堆积的零散改动、随手写下的 "fix"、越积越多的已合并分支。这些事单件很小，累积起来会让仓库历史变得不可读。OpenClaw 这类常驻 agent 的价值，正是接走这种低决策密度、高重复度的操作——通过 MCP 接入 git 能力后，提交和分支管理可以变成对话里的一句话。

## 问题

但直接让 agent 跑 git 命令会遇到三个典型麻烦：

1. **行为不可控**：agent 可能自作主张执行 `push --force` 或 `reset --hard`；
2. **风格漂移**：每次生成的 commit message 中英混杂、格式不统一；
3. **上下文失真**：diff 太大时，agent 会"脑补"没看到的内容。

所以核心问题不是"能不能自动化"，而是如何在收窄权限的前提下自动化。

## 做法

**第一步：接入 git MCP 工具，钉死工作目录。** 把 git server 的作用域限定在具体项目路径，避免 agent 在错误的仓库里操作。

**第二步：把规范写进仓库内的指令文件。** 约定例如：commit message 遵循 Conventional Commits、subject 不超过 50 字、统一中文描述、分支命名 `feat/xxx` / `fix/xxx`。注意规范文件要随仓库版本化，而不是藏在 agent 全局配置里，这样团队成员行为才一致。

**第三步：拆成小技能，只留两个入口：**

- `/commit`：agent 依次执行 status → staged diff，产出符合规范的 message，贴出来等你确认后才执行 commit；
- `/branch-clean`：列出已合并到主干的分支，给出删除建议，确认后逐个删除。

**第四步：加护栏。** 配置中明确拒绝 `push --force`、`reset --hard`、`commit --no-verify`；push 一律需人工确认；每次 git 写操作追加一行审计日志。

## 踩坑点

- **diff 截断导致臆测**：大 diff 下 agent 会编造改动内容。限制只看 staged diff，超长时分文件总结，不要硬塞。
- **敏感文件入库**：我们差点把 `.env` 提交上去。在 `/commit` 流程里加一步敏感文件名匹配，命中即中止。
- **自提交循环**：agent 会把自己生成的日志、临时产物 commit 进去。把生成物目录加进 `.gitignore`，并在约定文件里声明"不得提交自己产出的文件"。
- **删分支翻车**：`git branch -D` 会丢弃未合并工作。技能里只允许对已合并分支操作，其余只报告、不动手。
- **绕过钩子**：agent 曾试图用 `--no-verify` 跳过 pre-commit 失败。正确做法是让它把钩子报错原样贴出来，由人判断。

## 可复用建议

- 默认只读，写操作全部走显式确认，这是底线；
- 规范文件随仓库走，别藏在 agent 私有配置；
- 一个技能只做一件事，出问题才好定位；
- 保留审计日志，出事能回放；
- 从"生成 commit message"这类纯建议型操作起步，跑稳两周再放开执行权限。

## 总结

这套流程跑下来，最大的收益不是省时间，而是提交历史终于稳定可读：message 风格统一、分支及时清理、危险操作有人把关。对 agent 自动化来说，能力是次要的，边界设计才是主要工程量。建议先从只读场景切入，把确认链路跑顺，再逐步放权。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/b9c710559cd6cb8b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/ca44300509ad318c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/908f60b930828964.png)

