---
title: 跨平台消息路由实战：一个 Agent 同时值班 Telegram 和 Discord
feedId: 44199
source: 综合讨论
publishedAt: 2026-11-12
---

## 背景

我们的场景很典型：一部分用户在 Telegram 群里提问，另一部分常驻 Discord。最早是两个 agent 各挂一边，结果 prompt 改一次要同步两份，记忆和工具配置各一套，时间长了行为明显分叉。于是收敛成一个 Agent 同时服务两边，中间加一层消息路由。这篇记录做法和踩过的坑。

## 问题

直接在 Agent 里各接一个平台 SDK，很快会碰到四类问题：

- **消息模型不一致**：Telegram 有 reply_to 和 forum topic，Discord 有 channel 和 thread，语义对不齐，回复就会散落各处。
- **输出格式不一致**：Telegram 的 MarkdownV2 转义苛刻，Discord 的 markdown 是另一个子集，让 Agent 直接输出平台格式必然翻车。
- **限制不同**：Telegram 单 chat 约 1 msg/s、全局 30 msg/s；Discord 按 bucket 限速、单条 2000 字符（Telegram 是 4096）。
- **会话归属**：两边 chat id 都是数字，不区分平台必然撞 key。

## 做法

核心思路是三层：适配器层、路由层、Agent 核心。在 OpenClaw 里我们落成两个 channel 插件加一个路由中间件，Agent 核心不 import 任何平台 SDK。

**1. 统一消息信封。** 入站适配器把各自事件规整成同一个结构（示意）：

```json
{
  "trace_id": "m_01H...",
  "platform": "discord",
  "chat_id": "123456",
  "thread": "987654",
  "user": { "pid": "335", "name": "kx" },
  "text": "查下昨晚的构建日志",
  "attachments": [],
  "reply_to": null
}
```

**2. 会话路由。** session_key 统一为 `{platform}:{chat_id}:{thread ?? main}`，同一个 Agent 进程按 key 取/建会话，记忆与上下文自然共享。是否按平台再做用户身份映射看业务，我们目前选择共享。

**3. 出站渲染与回包。** Agent 只产出中性 markdown 加目标信息，渲染器按平台转格式、按字符上限切段，再交给对应适配器发送。回包默认走来源（origin），从哪来回哪去。

**4. 主动推送单独处理。** 定时任务和 webhook 触发没有"来源"，我们维护一张目的地表，显式指定 platform + chat + thread。

## 踩坑点

- **session_key 忘了带 platform 前缀**，测试环境两平台 id 撞车，排查一晚才定位。这个约定要尽早写进文档。
- **流式回复边生成边 edit message**，Discord 很快撞 edit 限速。改成缓冲 800ms~1s 合并更新，或只发一条占位加最终结果。
- **连接方式**：Telegram 的 webhook 与 long polling 二选一，本地调试 webhook 得挂隧道；Discord 记得在开发者后台开启 message content intent，否则收到的 content 恒为空。
- **事件幂等**：两平台断线重连都会重推事件，不按 message id 去重会出现重复回复。
- **线程能力不对齐**：Telegram 基础群没有 topic，我们退化为普通 reply。平台差异全部收进一张 capability 表，路由与渲染按表降级，不要散落 if-else。

## 可复用建议

- 适配器接口收敛为三个方法：`normalize_in`、`render_out`、`capabilities`。新增平台只写适配器，核心零改动。
- 信封必带 trace_id，日志按 trace 串链，一眼对上"哪条入站产生了哪条出站"。
- 把真实事件录成 JSONL 做回放回归，改路由后重放一遍，比在两个平台手工点快得多。
- 每平台单独记指标：入站延迟、出站失败、限速命中数。限速命中是路由健康度最敏感的信号。

## 总结

适配器 + 统一信封 + 会话路由解决了九成问题，剩下一成是各平台怪癖，靠 capability 表和降级策略兜住。收敛后最大的收益不在代码量，而在维护面：prompt、记忆、工具只有一份，两个平台上的 Agent 行为终于一致。后面要接第三个平台，理论上新写一个适配器就够了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/bcedad2b049d6d6a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/7f10437020c2e4ef.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/f6b85f5bff8ac12d.png)

