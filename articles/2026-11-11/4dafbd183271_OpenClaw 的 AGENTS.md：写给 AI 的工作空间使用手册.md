---
title: OpenClaw 的 AGENTS.md：写给 AI 的工作空间使用手册
feedId: 44066
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景

README 是写给人的，AGENTS.md 是写给 agent 的。OpenClaw 每次会话都会把工作空间里的 `AGENTS.md` 注入上下文，agent 怎么用目录、什么时候落盘记忆、敢不敢动 git，很大程度上由这份文件决定。它源自社区通行的 agents.md 约定，Codex、Cursor 等工具也认这个文件——也就是说，你为 OpenClaw 写的这份手册，换个 agent 指向同一个目录基本还能用，算是一份跨工具资产。

## 问题

最常见的状态是：装完 OpenClaw，模板里的 AGENTS.md 原样躺着，规则全靠聊天里口头交代。结果是每次会话都要重复"提交前先给我看 diff""别动 workspace 外面的文件"，换个会话就清零；heartbeat 自己加戏，memory 里堆满一次性上下文，agent 行为像开盲盒。根因只有一个：工作空间没有说明书。

## 做法

1. **打开工作空间根目录的 `AGENTS.md`，先读一遍默认模板**，删掉用不到的，补上你自己的约束。
2. **按节组织，规则写成可判定的祈使句**。一个够用的骨架：

```markdown
## 环境
- 工作空间：~/openclaw/workspace，读写默认限制在此目录内
- 回复用中文；commit message 用英文

## Git
- 不主动 push；push 前列出改动等待确认
- 不执行 force push

## 记忆
- 长期事实写入 MEMORY.md，一条一行，带日期
- 一次性上下文不落盘

## Heartbeat
- 只处理 HEARTBEAT.md 列出的检查项，异常时报告，不自行"修复"
```

3. **建立"纠偏即沉淀"的习惯**：同一件事口头纠正超过两次，就把它写进 AGENTS.md，而不是再说一遍。

## 踩坑点

- **越写越长**。整份文件都要进上下文，几百行的手册等于稀释关键指令。控制在几十行、只放高频规则。
- **指令不可执行**。"操作要小心"没有意义，"不主动 git push"才有判定标准。凡是没法回答"agent 怎么知道违反了这条"的规则，都是废话。
- **和其他文件职责重叠**。AGENTS.md 管流程和规矩，SOUL.md 管人格语气，TOOLS.md 管具体工具的笔记，MEMORY.md 是数据不是规则。混着写，改哪份都说不清。
- **只加不删**。工作流变了，旧规则会变成互相矛盾的噪声，定期清理失效条目。
- **把它当权限系统**。AGENTS.md 本质是提示词的一部分，是"软约束"。真正不能碰的东西，要靠 OpenClaw 的工具权限、沙箱配置去硬性兜底，文件里只做提示。

## 可复用建议

- 把 AGENTS.md 纳入 git（它本来就在工作空间里），改动可追溯，也方便多机同步。
- 团队内维护一份共用模板：环境约定、Git 规矩、记忆策略这几节基本通用，个人偏好单独放。
- 排查 agent 行为异常时，先重读 AGENTS.md 再怀疑模型——多数"不听话"是说明书没写清楚，或者写着互相打架的两条规则。
- 改完文件重启会话再观察，别在旧上下文里验证新规则，那样测不准。

## 总结

在 OpenClaw 的所有配置里，AGENTS.md 大概是杠杆率最高的一个：十几分钟的一次性投入，换来 agent 行为的长期稳定。它的定位是文档而非代码，不会一次写对，价值来自持续迭代——每次你想说"以后别这样"，先想想能不能写进文件里。手册写得越像给新人看的入职指南，agent 就越省心。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/b3063be162fd2d24.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/cb438a5d2695b314.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/2658d94eb81b866b.png)

