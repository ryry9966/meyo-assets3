---
title: 闹钟还是巡检员？OpenClaw 里 cron 与 heartbeat 的选型思路
feedId: 43709
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

OpenClaw 给 agent 提供了两条"定时"路径：**cron 任务**和 **heartbeat 心跳**。新手常见的两种误用：用 heartbeat 模拟定时播报（结果刷屏），或用 cron 写"看看有没有事做"的模糊任务（结果空转烧 token）。这篇帖把两者的差异和选型思路捋一遍。

## 本质区别

- **cron 是闹钟**：时间驱动，到点触发一个隔离会话，执行 prompt 里写死的任务，结果可以 announce 到频道。
- **heartbeat 是巡检员**：间隔驱动，按 `heartbeat.every`（默认约 30 分钟）向主会话注入一次心跳提示，由 agent 自己判断有没有值得做的事，没事就保持沉默。

三个关键差异：

1. **会话**：cron 每次跑隔离新会话，不带主会话上下文；heartbeat 跑在主会话里，能看到最近的对话和记忆。
2. **决策权**：cron 执行你写好的指令；heartbeat 把"要不要做"交给 agent。
3. **精度**：cron 支持 cron 表达式，精确到分钟；heartbeat 只有固定间隔。

## 选型：三问

1. 有没有明确时刻？每天 9 点日报、每周五清理 → **cron**。
2. 是不是"检查是否有事要做"？盯文件变化、扫未读、巡检 inbox → **heartbeat**。
3. 依赖主会话状态吗？要引用之前的对话/记忆 → heartbeat；不需要 → cron 的隔离会话更干净。

## 做法

**cron**（命令以你版本的实际 CLI 为准）：

```bash
openclaw cron add \
  --name "daily-digest" \
  --cron "0 9 * * *" \
  --prompt "汇总 garage 项目昨天新增的未决事项，不超过 10 行" \
  --announce "#daily"
```

要点：prompt 必须自包含——隔离会话里 agent 只知道 prompt 内容，输出格式和长度也要写死。

**heartbeat**（`openclaw.json`）：

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "45m",
        "prompt": "检查未读消息和 TODO 文件；只有需要我行动时才回复，否则保持沉默。"
      }
    }
  }
}
```

要点：heartbeat prompt 必须明确"无事则沉默"，否则 agent 每次心跳都会说话。

## 踩坑点

1. **时区**。网关跑在 UTC 的 VPS 上，`0 9 * * *` 是北京时间 17 点。给网关进程统一设 `TZ=Asia/Shanghai`，部署后先跑一次性任务验证触发时间。
2. **heartbeat 空转**。prompt 写成"看看有什么可以帮忙的"，agent 会每次都找活干。改用否定式指令："没有明确需要行动的事项就不要回复"。
3. **隔离会话缺上下文**。cron 任务里写"昨天聊的那个方案"，agent 是不知道的。路径、项目名、文件位置直接写进 prompt。
4. **群里刷屏**。heartbeat 输出到群频道，每次心跳都可能变成一条消息。heartbeat 目标设为私聊或静默，cron 的 announce 才适合进群。
5. **本地休眠**。网关跑在会休眠的笔记本上，错过的 cron 不一定补跑（行为取决于版本），heartbeat 也会漏拍。网关放常开设备。
6. **频率过高**。heartbeat 调到 5 分钟一次，token 消耗会很明显。巡检类任务 30–60 分钟足够。

## 可复用建议

- 分工口诀：**cron 管"到点必做"，heartbeat 管"没事别吵"**。
- 组合用法：cron 每 6 小时拉一次外部数据写到固定文件；heartbeat 巡检该文件，发现新增再决定是否播报。拉取是确定的，"要不要打扰用户"留给 agent 判断。
- 上线后先观察一周运行日志再调频率，别凭感觉拍间隔。
- cron prompt 里写清失败行为，比如"拉不到数据就跳过，不要编造"。

## 总结

cron 和 heartbeat 不是竞争关系，而是两种时间观的工具：一个是确定的时刻表，一个是弹性的注意力循环。把确定性交给 cron，把判断权交给 heartbeat，你的 agent 才既守时、又不聒噪。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/9b3150fa4666f132.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/78301ae382cf7161.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/c0737a2ff02ad02c.png)

