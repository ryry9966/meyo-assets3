---
title: OpenClaw 会话隔离实战：让子 Agent 干活，别让它弄脏主会话
feedId: 43417
source: 综合讨论
publishedAt: 2026-11-04
---

## 背景

在 OpenClaw 里跑复杂任务时，常见的模式是主 agent 拆解目标，然后 spawn 若干子 agent 分头执行——检索资料、跑脚本、调 MCP 工具。子 agent 有自己的对话历史和工具调用记录，这些中间过程如果原样回流主会话，主上下文很快就会被塞满。

## 问题：什么叫"污染"

实际遇到的几种情况：

1. **上下文膨胀**：子 agent 一轮工具调用动辄几千 token，三四个子任务跑完，主会话的系统指令被稀释，主 agent 开始"忘事"。
2. **噪音回流**：子 agent 的报错重试、冗长日志全进了主会话，后续推理被无关细节带偏。
3. **记忆串味**：子 agent 与主会话共享记忆 namespace，一次检索任务写入的临时偏好，改变了主会话后续的行为习惯。
4. **权限外溢**：子 agent 默认继承主会话全部工具权限，一个只需只读查询的子任务，拿到了写操作能力。

## 做法

核心原则一句话：**子 agent 独立开 session，只把结构化结论交回主会话，工件落盘。**

以我们组的配置为例：

```json
{
  "spawn": {
    "session_id": "sub-{{task_id}}",
    "inherit_system_prompt": false,
    "tools": ["fs.read", "http.get"],
    "memory_ns": "scratch/{{task_id}}",
    "max_turns": 12,
    "return_protocol": "json_summary"
  }
}
```

逐条拆开：

1. **独立 session_id**：子 agent 的全部对话和工具输出留在自己的 session，主会话里只留一条"已派发任务 X"的记录。
2. **关闭系统提示词继承**：给子 agent 单独写一份精简提示词，只描述任务边界和返回格式。继承主提示词会让它表现得分身，而不是外包工。
3. **工具白名单**：按任务最小集授权，能只读就不给写。
4. **记忆隔离**：子 agent 写入独立 namespace，任务结束可整体丢弃。
5. **统一返回协议**：约定最后一轮只输出 `{"status", "summary", "artifacts"}`。文件和日志先落盘，主会话只拿结论和路径。
6. **生命周期兜底**：`max_turns` 加超时回收，任务结束销毁 session，transcript 归档备查。

## 踩坑点

- **把整段 transcript 当返回值**：最常见错误。子 agent 跑了 30 轮，最后一把全贴回主会话，等于没隔离。
- **共享工作目录**：两个子 agent 写同一个 `output/`，互相覆盖。按 task_id 建子目录即可。
- **递归 spawn 没上限**：子 agent 又开子 agent，树状爆炸。建议全局限制 spawn 深度 ≤ 2。
- **回收不及时**：失败任务没配超时，僵尸 session 一直占配额。

## 可复用建议

- 把返回协议做成 schema 校验，不合规直接打回重试，别指望子 agent 自觉。
- 主会话收到结论后保留"需要细节再拉取"的动作，让读取 transcript 变成显式选择而不是默认行为。
- 定期抽检几个子 session 的 transcript，确认隔离真的生效——不少"污染"其实是配置没刷新导致的。

## 总结

会话隔离的本质是**控制信息回流的粒度**：过程留在子 session，主会话只消费结论。配置本身不复杂，难的是把返回协议、权限、生命周期当成契约来维护，而不是依赖 agent 的"自觉"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/27f2e03b434cbcf7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/5962f4e75d22f99f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/7dc1205d33ee7dbb.png)

