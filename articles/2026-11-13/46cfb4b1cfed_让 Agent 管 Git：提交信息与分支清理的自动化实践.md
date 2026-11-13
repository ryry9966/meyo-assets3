---
title: 让 Agent 管 Git：提交信息与分支清理的自动化实践
feedId: 44323
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

日常开发里 Git 操作大量重复：写提交信息、切分支、清理已合并分支、准备 PR 描述。这些活不不难，但琐碎且容易因偷懒变形——commit message 全是 "update"、"fix" 的仓库谁都见过。OpenClaw 的 agent 天然具备执行 shell 命令的能力，把它接到 Git 上是低成本的自动化切入点。

## 问题

实际跑下来，直接让 agent "帮我提交" 会遇到三类问题：

1. **信息质量差**：agent 只看 `git diff` 生成的提交信息往往只描述"改了什么"，说不出"为什么改"。
2. **误操作风险**：`git add -A` 会把本地配置、构建产物甚至密钥一并暂存；agent 在错误目录执行命令更是灾难。
3. **分支失控**：让 agent 自由创建分支，两周后仓库里躺着十几个 `test-1`、`final-final`。

核心矛盾是：Git 的写操作边界很模糊，而 agent 恰恰不擅长把握模糊边界。

## 做法

**第一步：分层权限，而不是全放行。** 我在 agent 的工具配置里把 Git 命令分了三档：

- 只读（status / log / diff / branch -a）：直接放行，agent 可自由了解仓库状态；
- 受控写（add / commit / 切换或新建分支）：允许执行，但每次操作前后强制输出结果摘要；
- 禁用（push --force / reset --hard / clean / rebase）：配置层直接拒绝，不依赖 prompt 约束。

**第二步：把约定写进配置，不靠口头交代。** 在 agent 的 system prompt 里固化团队规范：提交信息用 Conventional Commits（type 限定 feat/fix/refactor/docs/chore），分支名统一 `feature/`、`hotfix/` 前缀。约定写死比每次重复叮嘱可靠得多。

**第三步：给足上下文再生成提交信息。** 让 agent 起草前按固定顺序读取：`git diff --staged` + 最近 10 条提交 + 当前分支名；如果接了 MCP 的 issue 工具，再附上关联任务描述。这样生成的信息能对齐"为什么"，而不是复述 diff。

**第四步：分支清理做成定期任务。** 用 OpenClaw 的定时任务每周跑一次：列出已合并到主干的本地分支 → 生成删除清单 → 等确认后执行。清理可以从"发现"自动化到"执行"，但确认这一步留着。

## 踩坑点

- **`git add -A` 是重灾区。** 现在要求 agent 只 add 明确列出的文件，提交前把 staged diff 贴出来过目。有一次它把 `.env` 暂存了，幸好卡在确认这步。
- **cwd 校验。** agent 偶尔会在错误的仓库目录执行命令。我在受控写操作前加了一条"先输出 `git rev-parse --show-toplevel`"的规则，对不上预期仓库就中止。
- **不要让 agent 解冲突。** rebase/merge 冲突看似它能处理，实际它会"顺手"做出语义上错误的选择。冲突场景一律回退到人工。
- **detached HEAD。** agent 在 checkout 特定 commit 后继续提交，提交会悬空。规则里明确：发现 detached HEAD 就停下报告。

## 可复用建议

1. 权限三档模型（只读/受控/禁用）可以套到任何 shell 自动化场景，不止 Git。
2. 破坏性命令不要靠 prompt 拒绝，要在工具层硬拦。
3. 生成类任务（提交信息、PR 描述）先 dry-run 输出草稿，人工点头再落盘。
4. 所有 agent 的 Git 写操作留一份日志文件，出问题能回溯。

## 总结

这套流程跑下来，提交信息质量和分支整洁度提升明显，但真正的收益是边界清晰：agent 负责机械部分（读 diff、起草、列清单），人保留所有不可逆决策（确认提交、执行删除、处理冲突）。自动化不等于放手，把模糊判断从 agent 手里拿走，它反而更可靠。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/db97c3c15593c2b8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/99db3cc166c4425d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/3280b5448180f424.png)

