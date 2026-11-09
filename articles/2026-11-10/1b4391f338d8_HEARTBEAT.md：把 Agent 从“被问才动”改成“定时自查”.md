---
title: HEARTBEAT.md：把 Agent 从“被问才动”改成“定时自查”
feedId: 43951
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景：Agent 默认是被动的

OpenClaw 的会话模型是事件驱动的：有消息进来，agent 才被唤醒。但有一类工作天然没人来"问"——磁盘快满了、证书三天后过期、收件箱积压、上周说好的跟进没人动。这些事不会触发消息事件，所以再强的 agent 也会一直睡着。

HEARTBEAT.md 就是为这个设计的：gateway 按固定间隔（默认 30 分钟）给主 agent 注入一个心跳事件，agent 被唤醒后先读工作区里的 HEARTBEAT.md，把它当运行手册执行；文件为空或没有有效清单项时，按约定快速跳过，基本不消耗 token。

## 问题：外挂方案把逻辑撕成三片

此前常见做法是 cron + 脚本 + IM bot 拼一套，问题是上下文分裂：脚本能发现"磁盘 91%"，但它不知道该通知谁、之前通知过没有、处理惯例是什么。逻辑散在 crontab、shell、bot 三处，改一次需求要动三个地方。心跳机制把"何时醒"交给 gateway，"醒着干什么"收敛到一个 Markdown 文件里，与 agent 共享同一工作区、同一套工具。

## 做法：三步跑起来

**1. 打开心跳。** 在 `~/.openclaw/openclaw.json`：

```json
{
  "agents": {
    "defaults": {
      "heartbeat": { "every": "30m" }
    }
  }
}
```

间隔支持 `"2h"`、`"15m"` 这类写法，设为 `0` 关闭。改配置需要重启 gateway；HEARTBEAT.md 本身不用，下一次心跳就会读新版。

**2. 在 agent 工作区（默认 `~/.openclaw/workspace/`）写清单。** 每条尽量是"条件 + 动作 + 去重规则"：

```markdown
# 心跳任务
- 若 ~/inbox/ 有 24h 内新增文件，汇总文件名发到 Telegram
- 检查 docker ps 是否有 Restarting 状态容器，有则报告容器名
- 每天首次心跳：统计昨日 agent.log 的 ERROR 条数
- 已完成或已失效的任务，处理后直接从本文件删除
```

**3. 验证。** 把间隔临时调成 `2m`，看日志里的心跳 tick 和 agent 行为，符合预期后调回。要让主动消息推到 IM 而非留在本地会话，需配 target，细节以你版本文档的心跳小节为准。

## 踩坑点

- **成本与间隔线性相关。** 文件非空时，每次心跳都是一次真实模型调用。日常 30m–2h 足够，10m 以内先算算月账单。
- **文件膨胀。** HEARTBEAT.md 每次全量被读，稳定的 SOP 应放进 AGENTS.md，这里只放动态清单，控制在十几行。
- **重复通知。** agent 没有天然去重，同一个问题会每次心跳都报一遍。要么让它处理后删清单项，要么维护一个状态文件记录"上次通知时间"。
- **别塞长任务。** 心跳适合"检查 + 短动作"，重活让 agent 调脚本跑，模型只负责解读结果。
- **多 agent 路径。** 每个 agent 有自己的工作区和 HEARTBEAT.md，改错文件是"为什么不生效"的头号原因。
- **权限收窄。** 定时自动触发的 exec 调用务必配 allowlist/沙箱，别让值班员拿着 root 钥匙。

## 可复用建议

- 清单三段式：触发条件、动作、去重规则，歧义越少行为越稳。
- 默认"发现与上报"，处置要人确认；高频低风险的确认过再逐步放开自动化。
- 心跳做分钟级监控就好，秒级告警请交给真正的监控系统，Agent 不是 Prometheus。
- 跑通的清单沉淀成团队模板，比口头约定可靠得多。

## 总结

HEARTBEAT.md 的价值不在"定时执行"——cron 二十年前就做到了——而在执行体是一个带着完整工具和上下文的 agent：能读你的文件、调你的工具、按你的习惯汇报。把它当成每半小时醒一次的值班员来写运行手册，而不是当成万能定时器，体验会好很多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/286eccff4e97d7e8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/08eff60f656b1c61.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/f0b8599d861239b8.png)

