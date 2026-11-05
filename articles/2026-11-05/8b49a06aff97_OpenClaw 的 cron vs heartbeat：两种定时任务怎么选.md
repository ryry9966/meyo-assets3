---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 43516
source: 综合讨论
publishedAt: 2026-11-05
---

# OpenClaw 的 cron vs heartbeat：两种定时任务怎么选

## 背景

OpenClaw 里有两套容易被混为一谈的"定时"机制：

- **cron**：用 cron 表达式触发，每次跑在一个**全新隔离会话**里，带 payload 进、带结果出，可以投递到指定渠道。
- **heartbeat**：按固定间隔（默认 30 分钟）往**主会话**注入一条心跳提示，agent 带着完整上下文检查一遍，没大事就按约定静默（HEARTBEAT_OK），有情况才主动说话。

两者定位不同，但很多新用户会把固定任务塞进 heartbeat，或把"巡检"类需求交给 cron，结果要么 token 烧得快，要么任务"失忆"。

## 问题

三类典型误用：

1. 把日报、定时抓取这类**固定任务**写进 heartbeat 提示词——时间不准，主会话还越堆越重。
2. 把"盯着仓库看有没有异常"写进 cron——每次都是无记忆的新会话，判断不了上下文。
3. heartbeat 间隔设成 5 分钟——一天几百次 LLM 调用，绝大多数只是为了说一句"没事"。

## 怎么选：三个判断问题

1. 触发时间能否精确表达为 cron 表达式？能 → **cron**。
2. 任务是否依赖对话上下文或最近状态？是 → **heartbeat**。
3. 产出是否需要落盘、投递、可审计？是 → **cron** 更合适（会话隔离，日志干净）。

## 实操步骤

**固定任务走 cron：**

```bash
openclaw cron add --name daily-report \
  --cron "0 9 * * *" \
  --message "汇总今日待办与昨日提交，写入 daily.md" \
  --deliver --channel <目标渠道>
```

payload 里把必要上下文写全，或者明确让它去读 workspace 里的文件——它不继承主会话记忆。

**环境感知走 heartbeat：**

- 间隔从 30 分钟起步，别一上来就压到 5 分钟。
- 心跳提示词收窄成一行级别："检查 X 文件 / 提醒列表，无事则静默"。只描述"检查什么"，不塞"干什么活"。

**混合模式（推荐）：**

cron 负责定点产出，写到固定状态文件（如 `state.json` / `today.md`）；heartbeat 只负责"读状态、发现异常就说话"。用文件做两者之间的状态桥，既准又省。

## 踩坑点

- **heartbeat 每个 tick 都是一次真实调用**，哪怕 agent 静默。间隔就是成本主开关，先跑一周看用量再调。
- **cron 会话没有主会话记忆**，别指望它"记得昨天聊了什么"；要记忆就落文件。
- **时区问题**：cron 跟随宿主机时区，服务器在 UTC 时你写的"早上 9 点"会跑错，部署前先核对。
- **长任务重叠**：上一次没跑完下一次又触发，给 cron 任务加超时或拉长间隔。
- **共享主会话队列**：heartbeat 和你正在进行的对话在同一条队列里，心跳里塞重活会卡住交互响应。
- **停机不补跑**：网关关着的时候 cron 错过就错过了，恢复后不会自动补触发，关键任务要有兜底。

## 可复用建议

- 一句话选型：**时间确定用 cron，条件确定用 heartbeat**；两者都不确定，就 cron 落盘 + heartbeat 巡检。
- 每个 cron 任务独立命名、独立投递渠道、输出落文件，方便事后审计。
- 心跳提示词当成系统配置来维护，改之前先在隔离会话里试跑。
- 不要用 heartbeat 模拟 cron——它给不了精确时间；也不要用 cron 模拟巡检——它给不了上下文。

## 总结

cron 是"闹钟"，heartbeat 是"管家看一眼"。把**定时**交给 cron，把**判断**交给 heartbeat，中间用文件传递状态。想清楚每类任务的触发条件是什么，两条机制各司其职，个人自动化的绝大多数场景就都能覆盖了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/8c56fd0a42df2f24.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/9e3fc7f6f4cb5538.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/d189a4229a3d1469.png)

