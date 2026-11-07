---
title: Git 自动化实践：让 Agent 接管提交与分支，边界怎么划
feedId: 43726
source: 综合讨论
publishedAt: 2026-11-07
---

# 背景

用 OpenClaw 跑 agent 快两个月后，我把两类工作交了出去：写 commit message 和管理本地分支。这两件事机械、重复、容错率相对高，适合交给自动化。跑了一个多月，记录一下做法和踩过的坑。

# 问题

- commit message 质量随心情波动，`fix`、`update` 满天飞；
- 分支开完忘删，本地堆几十个 stale branch；
- 但直接让 agent 跑 git 命令有真实风险：误提交 secrets、force push、在错误分支上 reset。

核心矛盾是：自动化程度越高，越要收敛权限。我的原则是——**高频只读操作放行，写操作分批放行，破坏性操作永远人工确认。**

# 做法

1. **收敛工具面**。不给裸 shell，而是通过 MCP 暴露一个 git 工具集，白名单只含 `status / diff / log / add / commit / branch / checkout`；`push`、`reset --hard`、`rebase`、`clean` 不在名单里，需要时人工执行。
2. **在 AGENTS.md 写死工作流规则**：
   - 提交前必须先跑 `git status` 和 `git diff --staged`，向用户列出"将要提交的文件清单"，等确认；
   - message 遵循 conventional commits，subject 一句话，正文解释 why 而不是 what；
   - 一次提交只做一个意图，发现混合改动就提议拆分。
3. **分支管理**：约定从 issue 生成分支名（`feat/short-desc-123`）；合并完成后 agent 提醒清理本地分支，删除前必须确认已 merge。
4. **hook 兜底**：pre-commit 跑 lint，commit-msg 校验格式。agent 一样会被 hook 拦住，这层不依赖它自觉。
5. **全量留痕**：agent 执行过的每条 git 命令写进会话日志，出问题能回放。

# 踩坑点

- **`git add -A` 是最大事故源**。早期它把 `.env` 和两个调试脚本一起暂存了，幸好在"文件清单确认"这步被拦下。教训：先补全 `.gitignore`，再把"列清单、等确认"写进规则，而不是在 prompt 里写一句"小心点"。
- **diff 太长会漏看**。几百行的变更，让它分模块总结而不是全文塞进上下文，必要时分批提交。
- **detached HEAD 上乱操作**。有一次它在 detached 状态直接 commit，提交"消失"了。现在规则强制：任何写操作前先确认当前分支非空且非 detached。
- **冲突解决不放心**。rebase / merge 出冲突一律停下来问人，AI 拍脑袋选 ours/theirs 的代价太高。

# 可复用建议

- **从只读开始**：先让 agent 只做 status/diff/log 的分析和提交建议，人来执行；跑两周建立信任，再逐步放开写权限。
- **把流程封装成 skill 或斜杠命令**（`/commit`、`/branch`），提示词收敛在一处，团队成员直接复用。
- **权限白名单放在 MCP 工具层，而不是提示词层**。提示词会被长对话稀释、被绕过，工具层的硬限制才可靠。
- 定期让 agent 做一次"分支体检"：列出未合并分支、超龄分支，输出清理建议，人只负责点头。

# 总结

AI 管 git 的价值不在"全自动提交"，而在把机械劳动拿走、把判断留下：message 它写，内容你确认；分支它建，删除它提醒。权限收敛 + 清单确认 + hook 兜底，这三层做完，风险其实比人手滑更低。自动化的目标不是取消确认，而是把确认点放在真正值得确认的地方。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/8ea99d9bb36b2d1a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/a881b41ee6475339.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/74ddb94b12943c09.png)

