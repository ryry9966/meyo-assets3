---
title: AI 助手的 USER.md：别让 Agent 每次都重新认识你
feedId: 43780
source: 综合讨论
publishedAt: 2026-11-08
---

## 背景

Agent 用得越深，一个尴尬越明显：它每天陪你写代码、跑自动化、接 MCP 工具，却每次开场都不知道你是谁。技术栈、命名习惯、部署环境、语言偏好，全靠会话里临时口述，或者干脆靠它猜。

多数 Agent 框架（包括 OpenClaw 这类个人网关）支持工作区文件自动注入上下文。MEMORY.md 管项目动态，AGENTS.md 管行为规则，而 USER.md 管的是最稳定的一层：**你是谁**。

## 问题

没有 USER.md 时，通常有三类损耗：

1. **重复自我介绍**。每开一个新会话，都得再说一遍"我用 pnpm、macOS、别用 any"。
2. **偏好靠猜**。Agent 默认给你完整详尽的解释、生成你根本不用的 boilerplate，风格在会话间漂移。
3. **自动化产物分裂**。定时任务、插件、MCP 工具链各自拿到不同的"你"，产出不一致。

有人把这些塞进 system prompt，但 system prompt 适合放行为规则，不适合放会过时的个人事实，而且不好 diff、不好跨链路复用。

## 做法

一份能用的 USER.md 大概长这样：

```markdown
# Identity
- 称呼：Lao Wang；中文交流；时区 UTC+8

# Environment
- macOS 15 / Node 22 / pnpm；编辑器 VS Code
- 部署：Docker Compose 自托管，无 K8s

# Preferences
- 先给结论，推导过程放最后；不要客套
- 代码默认带类型，注释只写"为什么"

# Boundaries
- 不引入新状态库；不直接操作生产库，只给 SQL
```

落地步骤：

1. 在 Agent 工作区新建 USER.md，按 **Identity / Environment / Preferences / Boundaries** 四段组织，控制在 50–100 行以内。
2. 确认注入链路：OpenClaw 会自动读取工作区文件；自建链路可以用脚本把它编译进 system prompt，或通过 MCP resource 暴露。
3. **只写事实，不写愿望**。"我用 pnpm"是事实，"请写优雅的代码"不是——后者属于 AGENTS.md。
4. 建立迭代机制：Agent 猜错时，当场把纠正写成一条事实追加进去，而不是只在对话里纠正完就完了。放 git 里，方便 diff 和回滚。

## 踩坑点

- **写成小作文**。Agent 从叙事段落里提取约束不稳定，短句、条目、可 grep 才是正解。
- **塞敏感信息**。密钥、内网地址、客户名别写——这个文件会随上下文进入模型，也可能随日志外发。
- **与项目级指令冲突**。AGENTS.md 和项目配置优先级更高，USER.md 只放跨项目通用内容，别在两处维护同一件事。
- **只写不改**。三个月前的技术栈会让它持续推荐你已弃用的方案，建议每季度 prune 一次。
- **写模糊事实**。"做过很多后端"这类描述只会诱导它编造相关细节，写具体、可验证的条目。

## 可复用建议

- 四段模板直接抄：Identity / Environment / Preferences / Boundaries，每条一行。
- 把"纠正 Agent"变成落盘习惯：错了 → 追加一条 → commit，比在会话里反复解释便宜得多。
- USER.md（稳定事实）与 MEMORY.md（项目动态）严格分层，不要混。
- 定个季度日历提醒做 prune——这个文件的生命周期管理，比文件本身更重要。

## 总结

USER.md 是典型的低成本高杠杆上下文工程：十几分钟的维护成本，换来所有会话、所有自动化链路对"你"的一致理解。判断它写得合不合格有个简单标准——**像配置文件，而不是日记**。短、事实、可版本化，Agent 就真的认识你了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/e7aa9052ff9473c1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/5faaa421ae8560ce.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/08d919ef44de082a.png)

