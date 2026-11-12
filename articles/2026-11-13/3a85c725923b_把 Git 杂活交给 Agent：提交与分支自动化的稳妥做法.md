---
title: 把 Git 杂活交给 Agent：提交与分支自动化的稳妥做法
feedId: 44256
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

我在开发机上常驻一个 OpenClaw Agent，带 shell 执行能力。用下来发现，Git 里最耗心力的不是写代码，而是杂活：认真写 commit message、保持分支整洁、每天回忆“我今天改了什么”。这些操作低风险、高频次、规则清晰，正好是 Agent 自动化的甜点区。

## 问题

实际痛点三个：commit message 退化成 "update"、"fix"；三四个仓库的工作区常年 dirty，忘了提交；合并完的本地分支越堆越多，批量删又不敢。

但更关键的是反面问题：直接把 Git 交给 AI 很危险——它可能把 `.env` 一起 add 进去、对无关改动执行 `git add -A`、甚至自作聪明地 rebase 或 force push。所以这套自动化的核心不是“让它多能干”，而是“让它守规矩”。

## 做法

原则一句话：**低风险高频杂活自动化，高风险操作不授权或强制确认。**

**1. 封装工具，不给裸 shell。** 我没有直接用现成的 MCP git 工具集——暴露面偏大——而是自己写了个白名单封装脚本，Agent 只能通过它操作：

```bash
# git-helper：白名单封装，所有调用写入审计日志
case "$1" in
  status|diff|log)          exec git "$@" ;;
  commit)                   exec git commit "$@" ;;  # 上层保证只带显式路径
  branch-list|branch-clean) exec /usr/local/bin/branch-clean.sh ;;
  *) echo "denied"; exit 1 ;;
esac
```

**2. 提交流程先 dry-run。** Agent 先读 diff，产出符合 Conventional Commits 的 message 草案和待提交文件清单；系统提示词里明令禁止 `git add -A` 和 `git add .`，只允许显式列路径。白名单路径内自动执行，清单外必须等确认。

**3. 分支自动化。** 建分支统一命名 `feat/<topic>-<日期>`，方便脚本识别；清理只删“已合入 main 且超过 14 天”的分支，执行前先跑 `git branch --merged` 核对。

**4. 定时巡检代替实时托管。** 用 OpenClaw 的定时任务每晚扫一遍仓库列表，产出“dirty 仓库 + 建议 commit”清单发给我，人来拍板，Agent 只执行。

## 踩坑点

- **中文文件名转义**：默认 `core.quotepath=true`，diff 里中文文件名显示成八进制转义，Agent 容易误判文件。全局设 `git config --global core.quotepath false`。
- **密钥进上下文**：让 Agent 读全量 diff，等于让它读 `.env`。封装脚本里用 pathspec 排除 `':!*.env*'`，别靠提示词兜底。
- **pre-commit hook 失败死循环**：lint-staged 报错后 Agent 会进入“改代码重试”模式，为过检查瞎改。规则里写死：hook 失败即停、上报，不许自行修复。
- **作者身份污染**：Agent 提交要用独立身份，message 里加 Co-authored-by trailer，否则以后跑统计全是机器人。
- **rebase 别给权限**：试过一次让它自主解决冲突，中途上下文丢了，仓库卡在中间态，收拾了半小时。rebase/merge 一律手动。

## 可复用建议

1. **最小权限**：封装脚本优于裸 shell，宁可“做不了”，也不要“不该做”。
2. **先出计划再执行**：所有写操作要求 Agent 先输出计划，顺手就是审计记录。
3. **全量记日志**：每次调用落盘到本地日志，出问题能回放。
4. **只自动化杂活**：commit message、分支清理、状态巡检这三件跑得很稳；push、merge、rebase 保留人工，或至少加确认开关。

## 总结

Git 自动化的增量价值不在快，而在一致性：message 不再敷衍、分支不再堆积、每天改了什么有据可查。真正决定能不能长期用下去的是护栏——白名单命令、显式路径、dry-run 确认、审计日志。让 Agent 做它擅长的（读 diff、写 message、照清单执行），总开关留在自己手里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/a6e7e8ec53fcaa82.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/9657efcac742023a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/98499a3a0d6fc5d9.png)

