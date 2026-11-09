---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力的工程实践
feedId: 43908
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

过去半年我们把一个 OpenClaw Agent 从 5 个 MCP 工具扩到 30 多个：CI 查询、日志检索、报表导出、工单操作……结果很典型——系统提示词从 3k token 膨胀到 15k+，每轮对话都在为用不到的能力付费，模型选错工具的概率也明显上升。

Skills 机制针对的正是这个问题：把能力从“常驻上下文”改成“按需加载”。常驻的只有一份极轻量的元数据（名称 + 一句“何时使用”的描述），只有当模型判断当前任务匹配时，才把完整指令和脚本拉进上下文。

## 问题：上下文是稀缺资源

三个实际痛点：

1. **成本**：30 个工具 schema 常驻，但 90% 的会话只用得到其中 3 个；
2. **准确率**：候选工具越多，注意力越分散，误选率随之上升；
3. **维护**：所有能力挤在一个巨型提示词里，改一行要回归测试全部场景。

## 做法

### 1. 最小可用结构

一个 skill 就是一个目录：

```
skills/
  pdf-report/
    SKILL.md        # 必需：frontmatter + 指令正文
    scripts/
      render.py     # 可选：可执行脚本
    references/
      template.md   # 可选：按需读取的长文档
```

SKILL.md 的 frontmatter 里，`description` 是触发的唯一依据，正文才是执行指令。

### 2. description 按触发器写

- 反例：`用于生成报告`——模型猜不出什么时候用；
- 正例：`当用户要求生成周报、月报或导出 PDF 汇总文档时使用。触发词：周报、月报、report、export pdf`。

### 3. 三级渐进披露

- 第 0 级：frontmatter 常驻，只占几十 token；
- 第 1 级：命中后加载 SKILL.md 正文；
- 第 2 级：正文引用的 references 和脚本按需读取。

实际链路：用户说“导出本月周报”→ 命中 skill → 读正文 → 发现模板在 `references/template.md` → 按模板渲染 → 调 `scripts/render.py` 出 PDF。正文控制在 200 行以内，细节全部外置。

## 踩坑点

- **description 太短太泛**：90% 的“skill 不触发”源于此。要写场景和触发词，不是写功能介绍。
- **正文塞满细节**：有人把整份 API 文档贴进 SKILL.md，一次命中吃掉 8k token，等于没做按需加载。细节进 references。
- **脚本硬编码路径**：skills 会被复制到不同工作区，脚本里用相对路径，并启动时探测解释器版本。
- **触发冲突**：两个 skill 的描述都含“导出”，模型会随机选一个。描述要互斥，拿不准就先分类再进 skill。
- **高频操作别做成 skill**：一天触发几十次的能力，直接注册为普通工具更省事，加载动作本身也有成本。

## 可复用建议

1. 一个 skill 一个职责，宁可拆成两个小的；
2. 能写成脚本的逻辑不要写成散文——脚本可测试、结果确定、更省 token；
3. skills 目录进 git，像 review 代码一样 review description；
4. 维护一个固定评测集（10 条典型 query），改完描述跑一遍验证触发率。

## 总结

Skills 的本质是把提示词工程变成文档工程：把“模型该知道什么”和“模型此刻需要什么”分开。常驻成本降到几十 token，长尾能力靠渐进披露补齐。我们的 Agent 改造后，系统提示词降到 4k，工具误选率下降明显。代价是多一层“写好触发描述”的维护工作——但这层工作本来就该有人做。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/5b2d1d192317d8c1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/fe411059b0cfece3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/6dd469d71de9a103.png)

