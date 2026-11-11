---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 44163
source: 综合讨论
publishedAt: 2026-11-12
---

## 背景

OpenClaw 跑在自己机器上，接消息渠道、调模型，是一个长期在线的助手网关。它扩展能力不靠改代码，而是靠 Skills：每个 Skill 就是一个 `SKILL.md`，用 YAML frontmatter 声明元信息，正文用自然语言写给模型的操作手册，执行时配合模型自带的工具调用。

目录有三层：随发行版内置的 bundled skills、放在 `~/.openclaw/workspace/skills/` 下的 workspace skills、以及通过 `openclaw skills install` 装进来的托管 skills。

## 问题

把所有工具说明全量塞进 system prompt，是很多人第一版 agent 的写法，代价很明显：

- 每个 session 都为用不到的能力付 token，常驻成本随技能数线性上涨；
- 指令越多，模型选择性遵循越严重，准确率反而下降；
- 技能依赖的 CLI 没装、环境变量没配，照样被加载，跑到一半才报错。

Skills 机制解决的就是这三件事：让能力按需进入上下文，且只暴露当前环境真实可用的部分。

## 做法与步骤

**1. 理解渐进式披露。** 会话开始时，模型只看到各 skill 的 name + description 摘要列表（几十个 skill 也只占几百 token）。模型判断某个 skill 与当前任务相关，才去读它的正文。加载权在模型手里，成本从"常驻"变成"按需"。

**2. 写一个最小 skill。** 新建 `~/.openclaw/workspace/skills/invoice-summarizer/SKILL.md`：

```yaml
---
name: invoice-summarizer
description: 当用户要求汇总本月发票并生成对账摘要时使用；依赖 pdftotext。
metadata:
  openclaw:
    requires:
      bins: ["pdftotext"]
      env: ["INVOICE_DIR"]
---
## 步骤
1. 转换 INVOICE_DIR 下本月 PDF 为文本
2. 按供应商汇总金额，输出表格
## 边界
超过 50 份时先分批处理
```

description 决定模型是否选中它，务必写成触发条件，而不是名词解释。

**3. 用 requires 做环境门控。** `metadata.openclaw.requires` 支持 bins、env、config 三类声明。网关启动时检查，任一不满足，该 skill 不会出现在摘要列表里——模型根本"不知道"它存在，也就不会幻觉出半可用的调用。

**4. 验证。** 跑 `openclaw skills list` 查看加载与门控状态，`openclaw doctor` 排查依赖，然后新开会话实测触发。

## 踩坑点

- **description 写成"这是什么"**：模型从不主动调用。改成"当用户要求 X 时使用；做不了 Y"这类条件式描述，触发率立刻不同。
- **两个 skill 职责重叠**：模型随机二选一，行为不稳定。要么拆清边界，要么合并成一个。
- **requires 通过了但运行时找不到二进制**：gateway 以 systemd/launchd 跑时，PATH 与你的登录 shell 不一致，检查通过的位置可能不是 skill 实际执行的位置。必要时写绝对路径，或在启动环境里修 PATH。
- **改了 SKILL.md 不生效**：老会话已注入旧摘要，必须新开会话验证。
- **正文写成百科**：正文一旦被选中会整段读入上下文。长文档考虑拆成多个小 skill，或让正文引用外部文件按需读取。

## 可复用建议

- 高频重复操作固化成 skill，而不是每次手写 prompt；
- description 用固定模板：「当用户要求 ___ 时使用。步骤是 ___。当 ___ 时不适用。」
- 把 requires 当环境自检清单，让 skill 列表真实反映部署状态；
- 团队内把 workspace skills 目录放进 git 仓库统一分发，或发布到 ClawHub 复用；
- 极少数需要常驻的（如全局回复风格规范），可声明 always 强制加载，其余一律按需。

## 总结

Skills 机制的本质，是把能力说明从"全量常驻上下文"改成"摘要常驻 + 正文按需 + 环境门控"。收益是三重的：token 成本可控、指令遵循更准、技能可用性在启动阶段就可见。对长期运行的助手网关来说，这不是锦上添花，而是技能数量增长后仍然可维护的前提。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/73f58b5e181fc05e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/7a02acb72e319017.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/0f70048bef4744df.png)

