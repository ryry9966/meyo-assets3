---
title: Git 自动化实践：让 OpenClaw Agent 安全地管理提交与分支
feedId: 44272
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

我平时在终端里挂着 OpenClaw agent 干活。用了一段时间后发现，真正高频的重复劳动不是写代码，而是 git 杂活：想一个规范的 commit message、切分支、合并后清理、给 PR 写描述。这些事单独看只要两分钟，但一天十几次，而且经常被偷懒成 "fix"、"update" 和一坨 WIP。

于是我把提交和分支的日常操作交给 agent，跑了三周左右，把权限和流程收敛成了现在这套做法。

## 问题

直接说“你帮我管 git”是不行的，实际会遇到三类问题：

1. **提交质量不可控**：agent 可能凭对话记忆写 message，不看真实 diff，出现“描述与改动不符”的幻觉提交。
2. **越权操作**：push、`reset --hard`、对 main 的 rebase，一旦 agent 自作主张，恢复成本很高。
3. **混入敏感内容**：`.env`、密钥文件被顺手 stage 并提交。

## 做法

**第一步：收窄工具权限。** 通过 MCP 的 git server 接入，只读操作（status / log / diff / branch -l）直接放行；写操作（commit、建分支、merge）逐次确认；push 默认禁用，需要时我在对话里显式放行那一条命令。

**第二步：写一份 git SOP skill。** 内容很短，核心四条：commit message 用 Conventional Commits 且只能基于 diff 生成；只提交已 stage 的文件；禁止 `--force`、禁止改写已推送历史；冲突一律停下问人。SOP 放在仓库里随代码走，同事 clone 下来行为一致。

**第三步：固定提交流程。** 先 `git status` + `git diff --stat` 看全貌，再分文件读关键 diff，生成 message 后先展示给我确认再执行。大改动让它按逻辑拆成多个 commit，每个只含相关文件。

**第四步：分支管理。** 开工前 agent 自动建 `feature/xxx` 分支；功能完成后基于 diff 生成 PR 描述；合并后提醒清理本地和远端已合并分支。这部分每周用 cron 跑一次巡检即可。

## 踩坑点

- **message 幻觉**：早期它按我们聊天的内容写 message，实际 diff 完全不是那回事。对策写死在 SOP 里：禁止引用对话内容，只能引用 diff。
- **一锅端提交**：它会把无关改动全部 stage 进一个 commit。对策：staging 分步做，SOP 明确“未 stage 的不碰”。
- **secret 险些入库**：有一次 `.env` 出现在 diff 里。现在提交前自动跑一遍 gitleaks，再加一层 pre-commit hook 兜底——agent 也要过和人一样的检查。
- **大 diff 爆上下文**：几千行的 diff 直接塞会失控。先 `--stat`，再挑核心文件看，其余引用统计结论。

## 可复用建议

- 权限按“读全开、写确认、push 默认关”三档设计，跑稳两周再逐步放开。
- 审计别省：把 agent 执行过的 git 命令落一份日志，出问题能回放。
- hook 是最后防线，commitlint、gitleaks、CI 对所有提交一视同仁，不区分人还是 agent。

## 总结

这套东西的价值不在“全自动”，而在分工：机械部分（写 message、开分支、清历史）交给 agent，决策部分（push、合并、改写历史）留给人。权限先收紧再放开，流程写进 SOP 而不是靠对话临时约定，基本就能安全落地。目前我这边约七成日常 git 操作由 agent 完成，返工率很低，值得试。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/4ad03da71968eeaa.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/a90359d8db41434c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/ebca5849bed531ce.png)

