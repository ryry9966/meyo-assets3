---
title: OpenClaw session 隔离拆解：子 Agent 为什么不会污染主会话
feedId: 43731
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

用 OpenClaw 跑稍复杂的任务，几乎必然走到"主 Agent 拆活给子 Agent"这一步：主会话负责规划和决策，子 Agent 负责检索、批量处理、跑脚本。早期子 Agent 的过程记录会回灌主上下文，跑两轮就撑爆窗口；现在默认是隔离的，但很多人只知其然。这篇把我们踩过的坑和验证方法整理一下。

## 问题：不隔离时会发生什么

- 子 Agent 的工具调用、中间输出全部进入主会话历史，token 消耗翻倍；
- 主 Agent 的"记忆"被子 Agent 的过程细节干扰，后续规划质量明显下降；
- 两个 Agent 同时写同一个文件、共用同一个浏览器登录态，出现互踩；
- 重试时旧的子会话上下文被带进来，行为漂移很难排查。

## 做法：三层隔离

**1. 会话层。** 每个子 Agent 启动时分配独立的 session id，拥有自己的上下文窗口；主会话只接收它的结构化返回（结论 + 产物路径），不接收过程记录。示意配置（字段以你所用版本为准）：

```json
{
  "subagent": {
    "session": "isolated",
    "return": "summary",
    "workspace": "scratch/<run_id>"
  }
}
```

**2. 工作区层。** 子 Agent 默认在 `scratch/<run_id>/` 下工作，写文件、落中间产物都在自己的目录里，结束后把需要的文件登记回主工作区。并发跑多个子 Agent 时，这一层尤其重要。

**3. 传递层。** 父到子只传"任务目标 + 必要事实 + 输出契约"，子到父只回结论。上下文单向流动，是整个隔离的核心。

验证方法很简单：run 启动后打印 session 列表，确认每个子 Agent 有独立 id；再对比主会话的 token 统计——如果子 Agent 的工具输出出现在主上下文里，说明隔离配置没生效。

## 踩坑点

1. 给子 Agent 的上下文太少，它会自由发挥编事实；给太多，隔离等于白做。经验是控制在"一屏以内"，用结构化字段而不是大段粘贴。
2. 有状态工具（浏览器、数据库连接）默认可能共享实例，需要显式声明 per-session 或加锁，否则隔离了上下文也隔离不了副作用。
3. 子 Agent 再 spawn 子 Agent 的深度不设上限，session 会泛滥。建议 depth ≤ 2，超过就回到主会话协调。
4. 重试策略要明确：失败重试是新建 session 还是续用旧的。续用会带入失败残留，我们默认新建。
5. 环境变量和 cwd 会被继承，子 Agent 可能因此读到不该读的配置，敏感项要在 spawn 时覆写。

## 可复用建议

- 把子 Agent 当函数设计：入参明确、出参固定 JSON（result / files / caveats）、副作用收敛到自己的 scratch 目录。
- session id 一定要打进日志，事后审计"谁在什么时候改了什么"全靠它。
- 并行拆分前先确认任务幂等，否则隔离做得再好，也会在共享资源上翻车。

## 总结

session 隔离的本质不是"多开几个会话"，而是**上下文单向流动 + 副作用各自收敛**。会话层解决记忆污染，工作区层解决文件互踩，传递层解决信息质量。三层都到位，主会话才能长期保持干净，复杂任务才敢放心往下拆。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/0633aecaf9397c01.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/8ba2a1e99fb9ed86.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/e328f67d77963411.png)

