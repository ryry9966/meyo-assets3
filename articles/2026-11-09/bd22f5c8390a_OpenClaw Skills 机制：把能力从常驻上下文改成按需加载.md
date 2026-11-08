---
title: OpenClaw Skills 机制：把能力从常驻上下文改成按需加载
feedId: 43871
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

OpenClaw 的 Agent 运行时默认会把所有可用能力塞进上下文：MCP server 的 tool schema、插件清单、内置指令。能力少时没什么问题，但接了四五个 MCP server、十几个插件之后，系统提示词迅速膨胀，真正干活的空间被挤占，模型选错工具的概率也肉眼可见地上升。

Skills 机制就是针对这个问题的：把「能力」从常驻上下文改成按需加载。

## Skills 是什么

一个 Skill 就是一个目录，最小结构：

```
skills/
  log-analyzer/
    SKILL.md
    scripts/
      parse_errors.py
```

SKILL.md 分两段：frontmatter 里的 `name` 和 `description`，以及正文（操作步骤、约定、脚本用法）。OpenClaw 启动时只把每个 skill 的 name + description 注入上下文，通常几十 token；当模型判断当前任务命中某个 description，才读取正文；正文里引用的脚本，直接用 bash 执行拿结果，脚本本身不进上下文。三层渐进加载，上下文开销和实际用到的能力成正比。

## 怎么写一个能被正确触发的 Skill

1. **单一职责**。一个 skill 解决一类任务，比如「分析 web 错误日志并归因 5xx 突增」，不要做成日志瑞士军刀。
2. **description 写「什么时候用」，不是「是什么」**。`用于分析 web 服务器错误日志、定位 5xx 报错来源` 远好于 `一个日志分析工具`。模型靠这句话决定是否加载，写得像检索 query 一样具体。
3. **正文控制在两百行以内**，只写流程和约定；确定性逻辑全部下沉到脚本。LLM 负责判断和串联，bash 负责精确执行。
4. **用真实任务验证触发**：开一个会话，用你日常的原话提问，看它是否加载。不触发就改 description，而不是往正文里堆内容。

## 踩坑点

- **description 太泛**：写了「处理日志」，结果所有沾边请求都触发，加载了又用不上，白花 token；太窄则永远不触发。建议一两句话，包含触发场景 + 输入输出特征。
- **把脚本贴进正文**：有人图省事把上百行 Python 写进 SKILL.md，等于把按需加载打回原形。脚本是文件，正文里只写「执行 scripts/xxx.py，参数为……」。
- **忽略工作目录**：脚本里用相对路径，换个会话目录就跑挂。脚本内部显式定位自己的位置，或约定 skill 根目录变量。
- **Skill 数量失控**：每个 description 都是常驻开销，五十个之后 metadata 本身就成了负担。定期删掉触发率为零的 skill，同类合并。

## 可复用建议

- 先做减法再上 skill：能把 prompt 里手写流程固化下来的，先固化；能变成脚本的更好。
- 把 skill 当代码管理：进 git、走 review，改 description 要像改函数签名一样谨慎。
- 观测触发率：日志里记录 skill 加载事件，两周一盘，没被触发的要么改写 description，要么删。

## 总结

Skills 机制的核心不是多一种插件格式，而是把上下文当成稀缺资源来经营：常驻的只有索引，细节按需进场，确定性逻辑走脚本。写过几个 skill 之后你会发现，真正难的不是机制本身，而是把「这个能力什么时候该被想起」压缩成一句话——这恰好也是提示词工程的本质。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/6bee342f7638486e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/7ba1db88d80f09aa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/7f7ca339edbd2c76.png)

