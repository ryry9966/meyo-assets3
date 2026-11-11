---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力的工程实践
feedId: 44160
source: 综合讨论
publishedAt: 2026-11-12
---

## 背景

Agent 跑一段时间后，最直观的问题往往不是模型不够聪明，而是 system prompt 越来越胖：工具说明、MCP schema、各种固定指令全堆进去，token 成本上去了，模型反而更容易选错工具。OpenClaw 的 Skills 机制就是针对这个问题设计的：把能力拆成独立单元，元数据常驻，正文按需加载。

## 问题

Prompt 膨胀具体带来三个后果：

1. **注意力稀释**。常驻指令越多，模型对每条的遵守率越低，多个工具职责相近时尤其明显。
2. **成本线性增长**。常驻上下文每一轮都计费，不管这轮任务用不用得到。
3. **维护困难**。改一条规则，得在几万字符的 prompt 里找位置，改动没有边界。

## Skills 是怎么工作的

一个技能就是一个文件夹，核心是 `SKILL.md`：

```markdown
---
name: weekly-report
description: 生成周报时使用。读取本周期 commit 和 issue，按固定模板输出。
---
（这里是具体操作步骤、模板、注意事项）
```

关键设计是**渐进式披露**：只有 frontmatter 里的 `name` 和 `description` 会进入 system prompt，充当“索引”；正文的操作细节，只在模型判断当前任务相关时才读入。技能目录下还可以放脚本和参考文档，正文里用相对路径引用。

技能有三个来源：workspace（`~/.openclaw/workspace/skills/`，自己写的）、随发行版内置的技能、以及从 ClawHub 安装的社区技能。`openclaw skills list` 可以查看加载状态，配置里支持按 agent 粒度 enable/disable。

## 落地步骤

1. 在 workspace 下建技能目录，写 `SKILL.md`。
2. `description` 按“什么时候用 + 做什么”来写——这是模型的路由依据，值得反复打磨。
3. 正文当 SOP 写：步骤、边界条件、失败时怎么处理，假设读者是不了解背景的新同事。
4. 需要执行动作就附脚本，比让模型每次现编命令稳定得多。
5. `openclaw skills list` 确认加载，然后发一条真实消息触发它，观察是否按预期读取正文。

## 踩坑点

- **description 写虚了，技能等于不存在**。“一个处理各种事情的通用技能”这种描述永远不会被触发；写得太宽泛又会乱触发，抢走其他技能的机会。
- **别把 Skills 当 MCP 用**。Skills 提供的是流程知识，MCP 提供的是强类型的工具调用接口。需要参数校验、鉴权的服务走 MCP；“怎么用好这些服务”的经验，沉淀成 Skill。两者是分层，不是二选一。
- **脚本依赖要自查**。ffmpeg 之类的二进制不在环境里，技能显示加载成功，一跑就废。上线前手动跑一遍依赖检查。
- **正文也别贪长**。按需加载同样占上下文，长参考材料拆成附属文件，让模型需要时再读，主文件保持精炼。

## 可复用建议

- 把团队里重复出现的 SOP 沉淀成技能：一次编写，所有 agent 复用，比每次往对话里贴 prompt 划算。
- 技能粒度宁小勿大，一个技能解决一类任务，靠组合覆盖复杂场景。
- workspace 用 git 管理，技能变更走 review，出问题可回滚。
- 定期 `openclaw skills list` 做审计，禁用不再用的技能，别让索引悄悄膨胀回去。

## 总结

Skills 机制的本质是上下文经济学：常驻的只有索引，正文按需付费。它和 MCP 是互补关系——MCP 给工具，Skills 给使用工具的知识。实践中，写好 description 就完成了八成的工作，剩下的交给真实任务去迭代。与其追求一次写完美，不如先跑通一个最小技能，再逐步演进。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/6a7bac49728c2f28.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/c111528a88ebf07c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/02e973c5eb2186c5.png)

