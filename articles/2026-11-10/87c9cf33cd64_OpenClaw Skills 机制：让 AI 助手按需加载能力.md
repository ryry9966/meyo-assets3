---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 44039
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景

OpenClaw 的能力来源除了内置工具和 MCP 服务，还有一套 Skills 机制。一个 Skill 本质上是一个文件夹，里面放一个 SKILL.md：frontmatter 写 `name` 和 `description`，正文写操作指引。它和 MCP 的定位不同——MCP 注册的是结构化工具（schema + 调用），Skill 提供的是程序性知识：某类任务该怎么做、按什么顺序、边界在哪。

## 问题

做过 Agent 的都知道上下文预算有多紧。如果所有能力一次性塞进 system prompt——每个工具的 schema、每份操作文档全文——会带来三个后果：

1. token 成本随能力数量线性增长；
2. 模型面对几十个工具时，选择准确率明显下降；
3. 启动慢，排障时很难判断"模型到底看到了什么"。

Skills 的核心设计是渐进式加载：会话启动时只注入每个 Skill 的 name + description（几十 token 级别），当模型判断某个 Skill 与当前任务相关，才去读完整的 SKILL.md。

## 做法

1. **建目录**。个人技能放 workspace 的 `skills/` 目录（随工作区同步），通用能力放 `~/.openclaw/skills/`。
2. **写 SKILL.md**。frontmatter 至少包含 name 和 description。description 是路由的关键——模型靠它决定是否加载，所以要写清楚"什么时候用"，而不是"这是什么"。
3. **正文写操作步骤，不写百科**。假设读者是模型本身：命令、参数、边界条件、失败时怎么办。长参考内容拆成附属文件，正文只引用路径，用到再读。
4. **可选：加 gating**。frontmatter 支持 `env` / `bins` / `os` 等条件。比如 Skill 依赖 ffmpeg，就声明 `bins: [ffmpeg]`，环境不满足时该 Skill 不会出现在候选列表。
5. **验证**。新开会话，用自然语言提需求，观察日志里是否触发了对应 Skill。没触发就回头改 description。

## 踩坑点

- **description 写成名词解释**。"这是一个处理 PDF 的技能"——模型很难判断何时该用。改成动作导向："当用户要求合并、拆分、压缩 PDF 时使用"。
- **正文塞太满**。SKILL.md 超过几百行，触发一次就是一次上下文冲击。细节拆到附属文件，按需读取。
- **Skill 和 MCP 职责重叠**。同一功能既有 MCP tool 又有 Skill，模型会在两者间摇摆。经验法则：有稳定 API 的走 MCP，多步骤、需要判断的流程走 Skill，Skill 内部可以调用 MCP 工具。
- **gating 静默失效**。声明了 `bins` 但 PATH 里没有，Skill 直接消失，容易误判成"模型没触发"。排障第一步先确认 Skill 是否在候选列表里。
- **把密钥写进 SKILL.md**。它会被读进上下文，还容易被提交进 git。密钥走环境变量 + `env` gating 声明。

## 可复用建议

- 把 description 当路由规则写：动词开头，写清触发场景。
- 一个 Skill 只解决一类任务，宁可多个小 Skill，不要一个巨型 Skill。
- 确定性步骤封装成脚本放进 Skill 目录，正文只写"跑哪个脚本、参数是什么"——省 token，且执行更稳定。
- Skills 目录进 git，像 review 代码一样 review 每次修改。
- Skill 超过三四十个后，元数据本身开始膨胀，定期合并或归档低频 Skill。

## 总结

Skills 不是又一个插件系统，而是把"能力注入上下文"从全量加载改成按需加载。它的实际效果取决于两件事：description 的质量决定路由是否准确，正文的克制程度决定触发成本。我们团队把高频 SOP 从 system prompt 迁到 Skills 后，日常会话的基础 token 消耗降了约三成，工具选择的稳定性也明显改善。如果你的 Agent 已经出现"工具太多、prompt 太肥"的迹象，Skills 值得先试。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/07f05ac77704bce4.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/b35712ef85e18343.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/f061e1b513300279.png)

