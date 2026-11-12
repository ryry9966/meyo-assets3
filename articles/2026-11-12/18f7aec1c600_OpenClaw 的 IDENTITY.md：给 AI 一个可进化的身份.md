---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 44209
source: 综合讨论
publishedAt: 2026-11-12
---

## 背景

跑一个 7×24 常驻的 agent，头几周你会容忍它"通用助手"式的回答。但一旦它开始替你执行自动化、处理消息、调用 MCP 工具，一个稳定的人格就不再是装饰，而是接口——你和协作的人需要能预测它的行为边界。

## 问题：身份散落在哪

OpenClaw 把系统提示、SOUL.md、AGENTS.md、会话上下文分开管理。实际用下来最容易出乱子的就是"我是谁"这一层：

- 默认身份太泛，回答千人一面；
- 人格定义散在配置和提示词里，改一次要动多处；
- 没有变更记录，"上周它为什么变了"查不出来；
- 多 agent 共用一个 gateway 时，语气和自称会互相串。

## 做法

1. **定位文件**：workspace 根目录下的 `IDENTITY.md`，每轮会话注入上下文。没有就新建。
2. **写短**：只保留 name、creature（一个 emoji）、vibe 两三句、emoji 使用习惯、对用户的称呼。控制在 15 行内。示例：

```markdown
# IDENTITY.md
- Name: 阿钳
- Creature: 🦀
- Vibe: 直接、少废话，工程口吻；不确定就说不确定。
- Emoji: 回答里最多一个，讨论报错时不用。
- Address user as: 你
```

3. **用 git 管**：workspace 直接 `git init`，每次改身份单独 commit。身份变成可 diff、可回滚的资产。
4. **留审批门**：在 AGENTS.md 写明"agent 可提议修改 IDENTITY.md，确认后写入"。让它积累"我这样工作更顺"的观察，由你把关。
5. **定期修剪**：每两周删掉过时的描述，把真正稳定下来的行为固化进去。

## 踩坑点

- **写长了等于给每次会话加税**。IDENTITY.md 进每条上下文，40 行的"人设小作文"会稀释指令权重，也烧 token。
- **和 SOUL.md、AGENTS.md 冲突**。三处都定义语气，模型会在中间摇摆。约定：价值观进 SOUL，操作规则进 AGENTS，风格进 IDENTITY。
- **会话中途改文件不生效**。上下文在会话开始时快照，改完要新开会话或 `/reset`。
- **放开自改是漂移的开始**。没有审批门的自我进化，两周后你会得到一个陌生的 agent。
- **别写敏感信息**。workspace 可能被备份、同步或导出，放姓名和称呼习惯就够了。

## 可复用建议

- 把身份当代码：模板化（name / creature / vibe / 称呼四段），变更走 commit，重要改动写一行 changelog。
- 一个 agent 一个 workspace，不要共享身份文件。
- 检验标准不是"它像不像人"，而是"换一个会话，行为是否可预测"。

## 总结

IDENTITY.md 的价值不在"有人情味"，而在把人格从提示词里抽出来，变成一个可版本化、可审查、可演进的文件。进化要慢，审批要严，文件要短——这三条守住，长期运行的 agent 才不会慢慢变成你不再认识的东西。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/f96408b05a1985ff.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/78c20fb54802bb01.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/5255dca069e27f4f.png)

