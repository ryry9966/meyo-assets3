---
title: OpenClaw Skills 机制：让助手按需加载能力，而不是把说明书全塞进上下文
feedId: 43738
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

用 Agent 做自动化久了，能力清单会越攒越长：网页抓取、PDF 解析、数据库查询、定时任务……OpenClaw 的 Skills 机制就是为解决这个问题设计的。形式上它很简单——一个包含 `SKILL.md` 的文件夹，核心思想是**渐进式披露（progressive disclosure）**：

- 会话启动时，只把每个 skill 的 `name` + `description` 两行元数据注入上下文；
- 当任务命中某个 skill 时，agent 才去读完整的 `SKILL.md` 正文。

也就是说，装 30 个 skill 的固定成本，只有 30 条一句话的描述。

## 问题

在这套机制出现之前，常见做法是把所有工具文档全量写进 system prompt。后果有三个：

1. **token 成本线性增长**：每个 skill 几千字，十几个装下来系统提示词比对话本身还长；
2. **选择准确率下降**：工具描述互相干扰，模型容易在相似工具之间选错；
3. **实际利用率极低**：一次会话往往只用到一两个能力，其余全是死重。

## 做法

**1. 建目录结构。** 把 skill 放到 `~/.openclaw/skills/<skill-name>/SKILL.md`（或 workspace 的 skills 目录），frontmatter 里至少写 `name` 和 `description`。

**2. 把 description 写成路由函数。** 这是最关键的一步。不要写"用于处理文档"，要写清触发条件和反例，例如：

```yaml
description: 解析带表格的 PDF 并导出 CSV。当用户提到 PDF、发票、报表提取时使用。不要用于纯文本文件或图片 OCR。
```

触发词 + 排除项，模型的路由准确率会明显不同。

**3. 正文只写可执行内容。** 步骤、命令、边界条件、失败时的处理方式。超过几百行就该拆：长文档外链，脚本放同目录用相对路径引用。

**4. 验证与重载。** 用 `openclaw skills list` 确认 skill 被发现；改完 frontmatter 后执行 `/skills reload` 或重启 gateway，否则改动的描述不会生效。

## 踩坑点

- **description 太窄**：skill 永远不触发，你以为装了，实际模型根本没看见。
- **description 太宽**：什么都往这个 skill 上路由，反而污染其他工具的选择。
- **Skills 是提示词，不是沙箱**。它指挥 agent 的行为，但不提供权限隔离——skill 里让跑的脚本照样跑在宿主机权限下。别指望用 skill 做安全边界。
- **职责重叠**：两个 skill 都声称处理"数据导出"时，模型会随机选。在描述里显式分工："A 用于结构化数据库，B 用于网页表格"。
- **多机部署**：skill 加载自运行 agent 的那台机器。本地改完记得同步到 gateway 所在环境。

## 可复用建议

1. **一个 skill 只做一件事**，复杂流程靠 skill 之间组合，不要写百科全书式的大 skill。
2. **用 git 仓库管理 skills**，symlink 到 skills 目录，多台机器版本一致，改动能回滚。
3. **路由测试**：直接问 agent"处理 X 你会调用哪个 skill、为什么"，能快速暴露描述写糊的地方。
4. **从内置 skill 起步**：把自带的当模板抄结构，比自己从零写快得多。

## 总结

Skills 机制的本质是把"能力声明"和"能力本体"拆开：声明常驻、本体按需加载。省下来的是 token，换来的是工具选择的准确率。而整套机制的成色，最终取决于你 description 写得够不够像一台可靠的路由器——触发词明确、边界清晰、反例给出。花在描述上的十分钟，比花在正文上的半小时回报更高。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/5531f054b6562a00.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/e51488d285e06eb3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/22edc92e4f2a9e30.png)

