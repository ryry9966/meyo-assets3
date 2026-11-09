---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 43959
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景：身份不该硬编码在 prompt 里

OpenClaw 的 workspace 里有一组随会话启动注入系统提示的 Markdown：AGENTS.md 管协作规则，SOUL.md 管性格与措辞，USER.md 记录用户信息，而 IDENTITY.md 只回答一个问题——这个 agent 是谁。名字、原型、emoji、头像，就这几行。

这个设计的价值在于：身份是一等公民的配置文件，而不是散落在模板里的字符串。它可读、可 diff、可回滚，也就可以进化。

## 问题：没有身份文件的三个痛点

- 多 agent 并跑时，几个实例在同一个群里发言，用户分不清谁是谁；
- 会话重置或换机迁移后，"我叫什么、什么形象"要靠重新解释，每次口径还不一样；
- 调好的名字和设定散落在历史对话里，本质上是把身份存进了"记忆"，而记忆是会被压缩的。

## 做法：20 行以内，一次到位

1. 进入工作区（默认 `~/.openclaw/workspace/`），新建或编辑 IDENTITY.md；
2. 只写四个字段：`name`、`creature`（原型设定，一句话即可）、`emoji`、`avatar`（指向工作区内的图片文件）。性格措辞留给 SOUL.md，别混写；
3. `git init` 把 workspace 纳入版本控制，首次提交当作 identity 的 v1；
4. 改完后 `/new` 开新会话（或重启 gateway）使注入生效。验证方式很朴素：直接问它"你是谁、你长什么样"；
5. 建立进化回路：每月或遇到实际摩擦时（叫错、混淆、形象不符）改一次，commit message 写清动机，git log 就是你的 changelog。

多 agent 场景下，每个 agent 有独立 workspace，各自维护一份 IDENTITY.md，用不同 emoji 和头像在渠道里做视觉区分即可，不需要在系统提示里写"被叫到 X 时请回答"这类补丁逻辑。

## 踩坑点

- 会话中途编辑文件不会热加载。排障时先确认是不是没开新会话，别急着怀疑 prompt 模板；
- 如果你给了 agent 工作区写权限，它可能"自己改自己"。identity 变更必须走 git，发现问题直接 revert；
- 字段越写越多是常见滑坡。一旦超过 20 行，多半是把 SOUL.md 和 USER.md 的内容塞进来了，拆回去；
- avatar 用相对路径指向 workspace 内文件，别引用外链，离线或迁移时会挂；
- 别把用户名、内网路径、密钥写进身份文件——这个文件会进入每一次会话的系统提示。

## 可复用建议

- 把 identity 当配置代码管：改动走 PR，人工 review 后合并，合并即对下一次新会话生效；
- 多实例提前定命名规范（比如"项目名-职能"），避免后期在群里撞名；
- 每次 memory compaction 或定期维护时顺手 review 一遍 IDENTITY.md，成本一分钟；
- 让 agent 有提案权、没有表决权：它可以建议修改自己的身份，但落盘必须由人提交。

## 总结

IDENTITY.md 便宜到容易被忽视，但它是 agent 长期稳定运行的地基之一：一份可 diff、可回滚、随实际摩擦缓慢进化的身份文件，比任何精巧的 prompt 技巧都更能让 agent 保持"始终是同一个它"。先写 20 行，提交进 git，然后允许它慢慢变。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/1812a3e766aca24f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/976cb2cbbdbf7a0a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/7ee7b736ac602c72.png)

