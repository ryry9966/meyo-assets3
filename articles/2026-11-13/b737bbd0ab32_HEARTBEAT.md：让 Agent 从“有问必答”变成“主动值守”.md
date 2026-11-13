---
title: HEARTBEAT.md：让 Agent 从“有问必答”变成“主动值守”
feedId: 44333
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

用 OpenClaw 一段时间后，我发现自己对 Agent 的用法停留在问答：打开会话、提问、拿结果、关掉。Agent 大部分时间是死的。其实 OpenClaw 内置了一个容易被忽略的机制——heartbeat：daemon 按固定间隔（默认 30 分钟，可配置）唤醒 Agent 一次，Agent 醒来先看 workspace 根目录有没有 `HEARTBEAT.md`。文件不存在或为空，这次心跳就是 no-op，成本几乎为零；有内容，就按清单执行。平台给了节拍器，写不写谱子是我们的事。

## 问题

被动问答有三个真实痛点：一是事后补救，磁盘满了、CI 红了，都是我先发现再转告 Agent；二是重复劳动，日报、摘要类任务每次都要重新交代上下文；三是 Agent 的“持续关注”没有出口，它知道的东西只躺在会话里，不产生动作。

## 做法

1. 在 workspace 根目录新建 `HEARTBEAT.md`，每行一个任务，写可判定的条件而不是愿望：

```markdown
# Heartbeat
- [CHECK] disk usage over 80% → alert
- [CHECK] CI failed on main → alert with link
- [DO] 09:00 前后汇总昨日 commits，发日报
```

2. 任务分级：`[CHECK]` 是轻量检查（读接口、看文件），`[DO]` 是会产生输出的动作。心跳里以 `[CHECK]` 为主。
3. 在文件头部写输出纪律：“无事不发消息，异常才汇报，汇报不超过三句”。这条比任务本身更重要。
4. 按负载调 heartbeat interval。我的配置是 30 分钟起步，跑稳后再考虑加密。

## 踩坑点

- **心跳不是 cron。** 时间感来自模型，“每 5 分钟检查一次”这类高频任务不要塞进来，重调度交给系统 cron；heartbeat 只做轻巡检。
- **成本随清单膨胀。** 每次心跳都是一轮真实推理，清单超过 10 行就该砍。我删掉过“顺手整理笔记”这种条目——它每次都触发一轮写作。
- **噪音问题。** 不加“无事不扰”约束时，Agent 每次醒来都汇报“一切正常”，三天后你就会 mute 它。
- **条件模糊导致行为漂移。**“关注一下项目动态”每次执行结果都不一样；“main 分支 CI 失败则告警”才稳定。
- **没有热更新。** 文件改动在下一次心跳才生效，改完别当场等结果。

## 可复用建议

- 用三段式模板组织清单：巡检（异常告警）、日报（定时产出）、维护（低频清理）。每周 review 一次，删掉连续两周没有触发价值的条目。
- 能用 MCP 工具做确定性判断的，别让模型“看一眼”。比如磁盘水位让 Agent 调脚本拿数字再比较，比“检查磁盘状态”可靠得多。
- 把 heartbeat 当“值班表”而不是“愿望清单”：一条任务 = 一个可判定条件 + 一个明确动作 + 一个输出上限。

## 总结

`HEARTBEAT.md` 的机制很小：一份常驻文件加一个固定节拍。但它把 Agent 的角色从“问答机”变成了“值守员”。工程上的关键是克制——清单短、条件可判定、输出有纪律。建议先从一条 `[CHECK]` 开始跑一周，再决定要不要给它更多事做。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/253d087fc5f6e93e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/438ea202fb4704af.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/1a870923a8f1ef74.png)

