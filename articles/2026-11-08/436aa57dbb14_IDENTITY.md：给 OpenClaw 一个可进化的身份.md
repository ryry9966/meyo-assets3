---
title: IDENTITY.md：给 OpenClaw 一个可进化的身份
feedId: 43777
source: 综合讨论
publishedAt: 2026-11-08
---

## 背景

OpenClaw 的工作区里有几个一加载就进系统上下文的 markdown 文件：AGENTS.md 管操作规范，SOUL.md 管性格底色，USER.md 管你是谁。IDENTITY.md 是其中最短的一个，通常只写四行：name、creature、vibe、emoji。多数人的做法是装好时填一次，之后再没打开过。

用了一段时间后我发现，这个文件的真正价值不在“初始化”，而在“维护”。它更像一份需要持续 review 的线上配置，而不是一次性表单。

## 问题

我踩过的三类典型症状：

1. **语气漂移。** 新会话里 agent 的说话风格和几周前对不上。回查才发现是我在某次聊天里口头纠正了几句，但没落回文件，下个会话又漂回去。口头纠正只活在一个会话里。
2. **身份污染。** 早期我把任务偏好、回复格式、常用工具全塞进 identity 字段，vibe 一栏写了三行。结果上下文变胖，而且这些内容本该归 AGENTS.md 管。身份文件和操作文件混在一起，改哪个都心虚。
3. **多 agent 撞脸。** 挂了定时任务的 worker agent 后，主 agent 和 worker 在群聊里语气完全一样，看不出谁在说话。

## 做法

我现在的维护流程，四步：

1. **保持极简。** 四个字段各一行，vibe 控制在一个短句，全文件不超过 10 行。身份是“它是谁”，不是“它该怎么干活”。
2. **分层存放。** 谁是它 → IDENTITY.md；怎么做事 → AGENTS.md；性格底色 → SOUL.md；我是谁 → USER.md。写之前先问一句“这条信息属于哪一层”，能挡掉大部分污染。
3. **进 git。** workspace 整体纳入版本管理，IDENTITY.md 每次修改都写清 commit 原因。几个月后回看 diff，基本就是一份 agent 性格变更史，可追溯也可回滚。
4. **定期回收。** 每两三周翻一遍近期会话，把反复出现的口头纠正提炼成一行，按层级写进对应文件。同一条纠正说过第二次，就必须落盘。

## 踩坑点

- **改完不立即生效是正常的。** workspace 文件在新会话加载时才进上下文，当前对话里改完不会立刻“换人”。想验证就开个新会话。
- **别把 IDENTITY.md 当第二个 AGENTS.md。** 我塞过工具列表进去，效果远不如放 AGENTS.md 直接，还白占 token。
- **别照抄别人的模板。** 社区流传的身份文件看着不错，但 vibe 和原作者的使用场景是绑定的。抄过来大概率与实际行为对不上，反而加剧漂移。
- **不要放个人事实和敏感信息。** 你的名字、时区、偏好归 USER.md，密钥永远不该出现在任何 workspace markdown 里。
- **emoji 慎选。** 它会被多个渠道用作头像和消息前缀，选个小尺寸下还能认出来的。

## 可复用建议

- 把 IDENTITY.md 当成一份 10 行以内的“身份接口”：稳定、少变、有版本历史
- 每个 agent 一份 identity，name 与 config 里的 agent 名保持一致，方便日志定位
- review 频率跟着使用强度走：重度使用两周一次，轻度一个月一次足够
- 每次修改只改一个意图，commit message 说清动机，让 diff 本身成为文档

## 总结

IDENTITY.md 的维护成本几乎为零，但它决定了你在所有渠道里面对的是不是同一个 agent。当成一次性表单，几周后你会得到一个面目模糊的助手；当成持续维护的配置，你会得到一个行为可追溯、可回滚的长期协作者。进化的不是 AI 本身，而是你对它的描述精度。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/1b4104c665ee13b7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/499f45600c361ff4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/30168f9b93b8ad58.png)

