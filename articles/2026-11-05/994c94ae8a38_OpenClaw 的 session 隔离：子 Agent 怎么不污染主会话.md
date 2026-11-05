---
title: OpenClaw 的 session 隔离：子 Agent 怎么不污染主会话
feedId: 43520
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

OpenClaw 的典型用法是主会话做调度和决策，把重活拆给子 Agent：读大文件、跑检索、批量执行工具调用。子 Agent 理论上有自己的上下文，听起来很干净，但实际跑长任务的主会话还是会被撑大、被带偏。问题不在于子 Agent 本身，而在于**回流的东西没管住**。

## 问题：三类典型污染

1. **全量 transcript 回流**。子 Agent 跑完，把工具调用明细、读进来的大段文件内容一并塞回主会话，主上下文瞬间膨胀。
2. **session 串扰**。子 Agent 默认继承了主会话的 session id 或共享 memory，主会话的历史对它可见，它返回的内容也夹带着不该有的上下文假设。表现为"子 Agent 知道不该知道的事"。
3. **文件系统串扰**。子 Agent 把中间产物写进共享工作区，主会话后来读到某份草稿，把它当成了最终结论。

## 做法：四步隔离

**第一步，session 生命周期显式化。** spawn 子 Agent 时指定任务级 session，结束即销毁，不继承历史：

```yaml
# .openclaw/subagent.yaml
session:
  scope: task            # 每个任务独立 session，结束销毁
  inherit_history: false
report:
  format: json           # 只回流结构化结果
  max_tokens: 800
workspace:
  scratch: runs/{task_id}/
memory:
  write: task            # 禁止写全局 memory
tools:
  deny: [session.append, config.write, memory.write_global]
```

**第二步，约束回流。** 只允许子 Agent 返回一份结构化摘要（结论 + 关键证据 + 产物路径），不回传工具明细和中间推理。给 `max_tokens` 设上限，否则摘要本身就是二次污染。

**第三步，工作区命名空间化。** 中间产物统一写到 `runs/{task_id}/`，主会话只读最终 artifact。并发跑多个子 Agent 时，各自目录天然不冲突。

**第四步，工具白名单收敛。** 用 `deny` 掐掉会改主会话状态的工具（`session.append`、`config.write`），子 Agent 只保留完成任务所需的最小集合。

## 踩坑点

- **忘设 `inherit_history: false`**，默认继承导致串扰，症状隐蔽，建议写进团队模板。
- **摘要不设上限**。见过子 Agent 返回 8k 字"摘要"，等于换了个方向污染。
- **子 Agent 返回里带临时绝对路径**，主会话后续引用时路径已失效。约定统一用 `runs/{task_id}/` 相对路径。
- **异常退出没清 session**，残留上下文被下一个任务复用。定期扫一遍 sessions 目录，清孤儿。
- **多个子 Agent 共用一个 scratch 目录**，互相覆盖文件，最后谁对都说不清。

## 可复用建议

把"子 Agent = 无状态函数"作为默认心智模型：输入是 prompt 和只读资料，输出是校验过的结构化结果，其余一切销毁。主会话只做调度与裁决，不做中间计算。在团队里固化一份 subagent profile（session 策略、工具白名单、回流 schema），新任务直接套模板，比每次临时拍脑袋省心得多。

## 总结

session 隔离的本质是控制信息流向：中间态不回流、共享资源命名空间化、生命周期显式化。管住这三件事，主会话才能在长任务里保持清醒。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/b75a2a73a81f96c3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/02edad3d2e01c97c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/7f77d1c645aafcad.png)

