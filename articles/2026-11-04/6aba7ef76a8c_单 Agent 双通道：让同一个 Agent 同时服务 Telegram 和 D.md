---
title: 单 Agent 双通道：让同一个 Agent 同时服务 Telegram 和 Discord
feedId: 43385
source: 综合讨论
publishedAt: 2026-11-04
---

## 背景

最初我只在 Telegram 上跑了一个 OpenClaw Agent，做个人提醒和脚本协作。后来团队协作搬到了 Discord，问题来了：同一个 Agent 的能力和记忆，能不能同时服务两边，而不是再克隆一份配置？

## 问题

最省事的做法是再起一个实例接 Discord。试过就知道毛病在哪：

- 两份进程、两份记忆，同一个问题在两个平台得到不同答案；
- 定时任务重复执行，token 成本翻倍；
- 排障要看两套日志，状态对不齐。

本质上，Agent 的身份应该唯一，通道只是 I/O。所以目标定为：单 Agent 进程，多通道适配器，会话按对话隔离，长期记忆共享。

## 做法

**1. 配置层挂双通道。** 在配置里同时声明两个 channel，各自填 token（字段名以当前版本文档为准，示意）：

```json
{
  "channels": {
    "telegram": { "botToken": "..." },
    "discord":  { "botToken": "..." }
  }
}
```

网关会为每个 channel 各起一个适配器，入站消息统一进同一条总线。

**2. 会话键带平台前缀。** 这是最关键的一步。session key 如果只用 chatId，两个平台的 ID 空间可能重叠，会出现 A 平台的消息写进 B 平台的会话。改成 `{platform}:{chatId}` 后天然隔离。注意：隔离的是会话上下文，不是记忆——Agent 在两个平台仍然是"同一个人"。

**3. 出站格式归一。** Discord 和 Telegram 的 markdown 方言不同，代码块、链接、表情写法各有差异。我在适配器层做 sanitize：Agent 统一输出内部格式，由各适配器翻译成平台方言。不要让 Agent 自己判断平台，它总会忘记。

**4. 主动消息显式路由。** 定时任务和 cron 通知必须指定目标通道，否则回复能回到发起会话，通知却不知道去哪。我给每类任务加了 `notify_channel` 字段，缺省回落到 Telegram。

**5. 队列与限速。** Telegram 全局约 30 msg/s，Discord 按通道 5 msg/5s。两条通道各挂一个出站队列，Discord 那侧更保守，避免群发时吃 429。

## 踩坑记录

- **Discord 的 MESSAGE CONTENT intent 默认关闭**，后台没勾时 bot 能上线但收不到正文，表现为"只回不读"。首次接 Discord 必查。
- **消息长度上限不同**：Telegram 4096，Discord 2000。统一按 2000 分块最稳，代码块分块时要重新闭合围栏。
- **同一 bot token 跑两份进程**，Telegram 的 getUpdates 会直接冲突，两边轮流收不到消息。单实例不只是架构洁癖，是硬约束。
- **附件链接会过期**：Discord CDN 链接有时效，落库前先转存，否则隔天回看会话全是死链。

## 可复用建议

1. 把 platform、chatId、userId 写进每条入站信封，Agent 和工具都从信封取上下文，别让工具自己去猜；
2. 适配器只做协议翻译，不做业务判断；业务逻辑全在 Agent 层，新接一个平台的成本就是写一个适配器；
3. 上线前用一个环回测试通道做 dry-run，把格式归一、分块、限速全过一遍，比在群里翻车便宜得多。

## 总结

双平台路由的难点不在"接进来"，而在会话身份、格式方言和主动消息的目标选择。架构上坚持单 Agent + 多适配器，把平台差异压死在适配器层，之后接 Slack、飞书都只是增量工作。这套结构目前稳定跑了一个多月，没有再动过——对一个消息基础设施来说，"没再动过"就是最好的评价。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/653b866294191e70.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/3e380bd563292642.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/02ed5364f8f4478f.png)

