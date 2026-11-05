---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 43548
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

OpenClaw 的 agent 每次会话都会读取 workspace 里的一组 Markdown：`AGENTS.md` 管操作规范，`SOUL.md` 管性格语气，`USER.md` 记用户信息，而 `IDENTITY.md` 回答一个最基础的问题——"你是谁"。它通常只有几行：Name、Creature、Vibe、Emoji，外加一句 Description。很多人初始化完就不管了，但跑了一段时间多渠道、多 bot 之后，我发现这是杠杆最高的一处配置。

## 问题

不认真管它，会遇到三类麻烦：

1. **自我指涉混乱**：群聊或多 agent 场景下自称摇摆，用户分不清在跟谁说话。
2. **身份与规则混写**：把"永远用中文""不许乱承诺"这类行为规则塞进来，文件越写越长，自述被稀释，还与 SOUL.md 重复、逐渐漂移。
3. **改动不可追溯**：改了身份但没记动机，行为变差时无法定位是哪次修改引入的。

## 做法

我的流程五步：

1. **字段固定**：沿用模板的 Name / Creature / Vibe / Emoji / Description，不自创字段，格式稳定注入效果才稳定。
2. **职责分离**：IDENTITY.md 只回答"我是谁"，Description 一句话供自我介绍、群聊签名等场景复用；语气与规则归 SOUL.md、AGENTS.md。整个文件控制在 10 行内。
3. **Git 版本化**：workspace 纳入 git，每次改身份单独提交，commit message 写清"为什么改"。身份变更也该有 code review 的纪律。
4. **让 agent 提案**：每两周让 agent 基于近期对话输出身份修订建议，只要 diff、不要整文件重写，例如"最近常被问部署问题，建议 Description 补一句擅长运维"。人工审阅后合并。
5. **回归验证**：合并后开新会话，用固定三问测："你是谁""一句话介绍自己""你和另一个 bot 什么关系"，确认自述符合预期。

## 踩坑点

- **改完不生效就以为写错了**：IDENTITY.md 在新会话加载，旧会话不会热更新，验证前先开新会话。
- **让 agent 全权重写自己**：我试过让它"优化"整份文件，结果语气越来越讨好、身份越改越泛。只收 diff、只由人合并，这条必须守住。
- **Description 写成小作文**：超过两句，自我介绍就开始跑题，宁短勿长。
- **多 bot 共用一份身份**：每个 agent 独立 workspace、独立 IDENTITY.md，群聊里的自称才不会串。
- **随手改字段名**：模板字段是解析约定，改成中文键或自创键可能导致注入异常。

## 可复用建议

- 把 IDENTITY.md 当 API 对待：**schema 稳定、内容精简、变更走 PR**。
- 固定迭代节奏（两周一次足够），拒绝拍脑袋随手改。
- 三问回归 + git 历史，让"身份为什么变成这样"永远可回答。
- 提案让 agent 出、决策由人做——这正是"可进化"与"身份漂移"的分界线。

## 总结

IDENTITY.md 的价值不在文件本身，而在它把"agent 是谁"变成了可版本化、可审阅、可回滚的工程对象。几行 Markdown，配上 git 和一套轻量修订流程，身份就能随使用持续进化而不失控。小文件，大杠杆，值得认真管。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/5098153bc6c7ed04.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/4a0f6f9324ba7856.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/5935c0b2fcfd467f.png)

