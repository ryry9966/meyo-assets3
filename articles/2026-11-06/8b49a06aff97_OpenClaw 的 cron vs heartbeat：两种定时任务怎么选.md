---
title: OpenClaw 的 cron vs heartbeat：两种定时任务怎么选
feedId: 43602
source: 综合讨论
publishedAt: 2026-11-06
---

# 背景

Agent 常驻跑起来之后，第一类真实需求几乎都是“定时做点事”：早上推日报、定期巡检磁盘、每周汇总依赖更新。OpenClaw 给了两条路：**cron job** 和 **heartbeat**。两者都能触发 agent，但设计意图完全不同。我们团队早期混用，要么 token 白烧，要么任务不准点，这里把选择依据整理一下。

# 两者分别是什么

**Cron job**：独立调度器。用 `openclaw cron add` 注册，标准 cron 表达式触发一次完整的 agent 会话。时间到 → 执行你给的 prompt → 按配置投递结果到频道，或静默落盘。

**Heartbeat**：agent 自身的心跳节拍。每隔 N 分钟（默认 30 分钟）醒来一次，读 workspace 里的 `HEARTBEAT.md`，有事就处理，没事回一个极简确认继续睡。

一句话区分：**cron 是“到点必须干的活”，heartbeat 是“醒来顺手看一眼的待办”**。

# 怎么选：三条判断标准

1. **触发条件是“时间点”还是“状态”？** 每天早八点发摘要 → cron。“有空检查下磁盘是否快满、依赖是否有更新” → heartbeat。前者关心时刻，后者关心条件成立。
2. **动作是否固定？** prompt 每次一样、输出格式可预期 → cron。动作需按当时情况自由发挥（多项待办、自我维护）→ heartbeat。
3. **频率多高？** 心跳是全局节拍，调到几分钟一次，每次唤醒都烧 token。真需要分钟级轮询外部状态，用 cron 或事件回调，别拿 heartbeat 硬扛。

# 做法与配置

Cron 典型注册：

```bash
openclaw cron add \
  --name "daily-digest" \
  --cron "0 8 * * *" \
  --message "汇总昨日会话要点，输出不超过 10 行" \
  --deliver --channel telegram
```

一次性任务用 `--schedule-at`；用 `openclaw cron list` 和 `openclaw cron runs <id>` 回看历史执行与失败原因。

Heartbeat：把周期性待办写进 workspace 的 `HEARTBEAT.md`，一项一行，写清“何时做、做到什么程度算完成”：

```markdown
- 每次心跳检查 /data 磁盘占用，超过 85% 才通知我
- 周一上午检查依赖更新，只汇总，不升级
```

间隔在配置里调：

```json
{ "agents": { "defaults": { "heartbeat": { "every": "45m" } } } }
```

新版本中 `HEARTBEAT.md` 为空会直接跳过唤醒，接近零成本；老版本建议任务做完手动清空文件。

# 踩坑点

- **时区**：cron 按进程系统时区解析，Docker 里默认常是 UTC，“早八点”实际跑在下午四点。容器里显式设置 `TZ`。
- **cron 静默失败**：不配 `--deliver` 时结果只落在本地，挂了也没人知道。重要任务务必配投递，或在 prompt 里要求失败时主动说明。
- **heartbeat 空转**：老版本无论有没有待办都完整唤醒一次。发现用量异常上涨，先查这里。
- **任务重叠**：心跳任务跑得比间隔还长时会叠加触发。长任务挪去 cron，heartbeat 只留轻量检查。
- **prompt 里写死时间**：cron 的 message 里写“今天”没问题，但 agent 不一定知道确切日期，让它在执行时自己取当前时间。

# 可复用建议

- 命名带节奏：`daily-*`、`weekly-*`、`watch-*`，三个月后还能看懂。
- 分工一句话：**cron 管对外输出，heartbeat 管对内巡检**。
- 心跳清单控制在 3 项以内，超了说明该拆成独立 cron 或事件。
- 每月跑一遍 `cron list`，清掉没人认领的僵尸任务。

# 总结

Cron 是确定性调度，heartbeat 是低成本的环境感知。按“时间点 vs 状态、固定 vs 灵活、频率高低”三问分配，绝大多数场景一秒定案：准点必做的事交给 cron，“顺手看一眼”的事写进 `HEARTBEAT.md`。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/02b7669a488240b3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/b71fa9f353152796.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/9f4db6392754ff93.png)

