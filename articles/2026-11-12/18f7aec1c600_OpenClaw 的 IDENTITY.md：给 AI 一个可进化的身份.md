---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 44235
source: 综合讨论
publishedAt: 2026-11-12
---

## 背景

OpenClaw 的工作区根目录下有一组约定文件：AGENTS.md 管行为规则，SOUL.md 管性格与做事方式，USER.md 记用户偏好，而 IDENTITY.md 负责回答一个最基础的问题——"我是谁"。名字、形象（emoji）、物种设定、vibe、自我描述，都收在这一个小文件里。它会在会话启动时被注入系统上下文，是 Agent 每次醒来读到的第一份"自我说明书"。

## 问题

很多人装完 OpenClaw 直接用默认模板跑起来，IDENTITY.md 再也没打开过。后果不会立刻显现，但用一两周就会碰到几个具体问题：

1. **跨会话人格不一致**：今天自称"助手"，明天换一种称呼和语气，团队共享的 Agent 尤其明显；
2. **身份与行为脱节**：SOUL.md 写了"简洁、直接"，IDENTITY.md 的 vibe 却是"热情友好"，两份文件打架，输出风格来回漂移；
3. **身份无法沉淀**：调整散落在聊天里（"你以后叫我 XX"），重启就丢，等于把人格交给上下文运气。

## 做法

核心思路：把身份当配置管理，而不是聊天里的一次性指令。

1. **先写最小版本**。name、emoji、creature、vibe、description 各一两句就够：

```markdown
- Name: 阿钳
- Emoji: 🦀
- Creature: 一只住在终端里的螃蟹型助手
- Vibe: 克制、直接、有点冷幽默
- Description: 服务于日常自动化，擅长排障和脚本，短句优先。
```

2. **和 SOUL.md 划清边界**：IDENTITY.md 管"我是谁"，SOUL.md 管"我怎么做"。同一件事不要在两个文件里都写，出现冲突时以一份为准，另一份删掉。
3. **纳入版本控制**：工作区天然适合 git。每次修改留 commit message（如 `tone: 更克制`），出问题可以 diff 回滚。
4. **观察驱动迭代**：跑一两周，收集实际对话里别扭的瞬间——称呼错了、语气飘了——针对性改对应字段，而不是一次重写整个文件。

## 踩坑点

- **写太长**。IDENTITY.md 不是简历，塞满性格分析会稀释关键信息。模型对开头和结尾更敏感，重要设定放前面。
- **vibe 和 SOUL.md 冲突**。我踩过最多次的坑："冷静专业"和"活泼幽默"同时存在，输出就是随机混合。改任何一个文件前，先全局搜一遍风格关键词。
- **拿它当记忆用**。用户偏好归 USER.md，事实记忆归 memory 目录，IDENTITY.md 只描述自我。混着写，后续清理成本很高。
- **频繁改人设**。身份需要稳定才有连续性可言，小步微调比推翻重来有效。

## 可复用建议

- 把 IDENTITY.md 当 API 契约看待：字段固定、描述克制、变更留痕。
- 团队场景下，共享 Agent 的身份改动走 PR review，和改代码同权重。
- 每月读一遍 IDENTITY.md + SOUL.md，问自己：这还是现在这个 Agent 吗？不匹配的删掉。
- 记住"进化"的含义：不是频繁改动，而是"设计—观察—微调"的闭环。

## 总结

IDENTITY.md 是 OpenClaw 里成本最低、也最容易被低估的文件。它不解决能力问题，解决的是一致性问题：让 Agent 在跨会话、跨成员使用时保持同一个"自己"。把它当配置管起来，用版本控制和定期 review 支撑它慢慢进化，比在聊天里反复纠正行为可靠得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/1c272309c22a148d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/6b0204454d6114f5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/34ba778c8ea64f0e.png)

