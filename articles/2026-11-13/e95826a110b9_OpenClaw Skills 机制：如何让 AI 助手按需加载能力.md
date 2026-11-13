---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 44304
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

在 OpenClaw 里给 agent 扩能力有三条常见路径：MCP 负责接外部工具，插件负责改运行时行为，而 Skills 解决的是另一个问题——**把"怎么做某类事"的流程知识，按需喂给模型**。

它的核心设计是渐进式披露（progressive disclosure）：agent 启动时只扫描每个 skill 的元数据（name + description），正文并不进上下文；只有当任务与描述匹配时，SKILL.md 的内容才被加载。skill 内部还可以再引用脚本和参考文档，形成"元数据 → 正文 → 资源"的三级按需结构。

## 问题

没有这层机制时，常见做法是把操作手册、格式约定、领域知识全塞进 system prompt 或身份文件。后果很直接：

- 上下文常驻膨胀，每轮对话都为用不到的知识付费；
- 注意力被稀释，真正相关的指令反而容易被淹没；
- MCP 工具装多了同理，工具 schema 本身就吃掉几千 token。

实测下来，几十个 skill 的描述加起来只有几百 token，比全量注入低一个数量级。这就是按需加载的意义。

## 做法

以给 agent 加一个"周报生成"能力为例：

1. **建目录**：workspace 下建 `skills/weekly-report/`，一个 skill 一个文件夹；
2. **写 SKILL.md**：frontmatter 填 name 和 description。description 是触发的唯一依据，写清"什么时候用"，例如"当用户要求汇总本周工作、生成周报或站会纪要时使用"，别写成抒情文案；
3. **正文只写流程**：步骤、字段约束、输出格式，控制在几十行以内；
4. **大块内容外置**：详细模板放 `references/`，确定性逻辑写成 `scripts/` 里的脚本（比如拉 git log、聚合 issue），正文注明"执行时读取"；
5. **验证**：重启 gateway 或等下一次 heartbeat 重扫，然后丢一个应触发它的任务过去，确认它真的加载了。

## 踩坑点

- **description 太泛**，skill 永远不被触发。用任务动词和对象词来写，把它当搜索引擎的广告位。
- **数量失控**。描述列表本身也占上下文，别给每个小偏好都建 skill，高频场景优先。
- **Skills 是提示词，不是沙箱**。scripts 以 agent 自身权限执行，密钥不要写进 skill，敏感操作在正文里写明需人工确认。
- **和身份文件抢话**。行为准则放身份文件，操作步骤放 skill，两处冲突时模型会随机站队。
- **改完没重扫**。以为没生效，其实是旧索引；先确认加载状态再排查。
- **触发范围重叠**。两个 skill 都匹配时 agent 会选错，把各自的边界条件写进 description。

## 可复用建议

- 命名用动宾结构（如 `pdf-extract`、`weekly-report`），一 skill 一件事；
- 确定性步骤交给脚本，判断性步骤写指令，别用提示词硬模拟计算；
- 持续迭代 description：遇到没触发或误触发的 case，回来改描述，通常比改正文更有效；
- skill 就是一个普通文件夹，git 管理即可，团队内复用成本几乎为零。

## 总结

Skills 的价值不在"能力多"，而在"按需"。把它当成给 agent 维护的一套操作手册库：元数据做索引，正文按需加载，细节再下钻到脚本和参考文档。上下文省下来的每一个 token，都应该留给当前任务本身。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/ed5aea94fd33cc2e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/87b569fa1ca2e8d1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/0b3fd9f0c32a41d7.png)

