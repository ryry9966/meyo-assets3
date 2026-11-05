---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 43560
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

我们在内部用 OpenClaw 跑日志清理和构建产物归档，最怕的不是 agent 干不了活，而是它“干得太起劲”——比如把 `rm -rf` 的路径算错一格。这篇帖拆解一下 OpenClaw 的 sandbox 安全模型，以及为什么一个拿到 shell 能力的 agent 大概率删不掉你不想让它删的东西。

## 问题：提示词不是安全边界

先说结论：在 system prompt 里写“请小心不要删除重要文件”是不可靠的。LLM 对路径的推断存在幻觉，相对路径、符号链接、通配符都是重灾区。我们自己遇到过一次险情：要求清理 `dist/`，agent 在某次重试中把上下文搞混，生成了一条作用范围大得多的命令。那一刻救场的不是提示词，而是沙箱根本没给它那个路径的写权限。

正确思路是：假设模型一定会犯错，让错误被结构和权限挡住。

## 做法：四层防线

**第一层：容器隔离（物理边界）。** sandbox mode 开启后，`exec` 和文件读写都在 Docker 容器内执行，默认只把 workspace 以 bind mount 挂进 `/workspace`，宿主机其余目录对容器不可见。模型就算生成一条天怒人怨的 `rm -rf /`，删的也只是容器里那个空根。

**第二层：工具策略（最小权限）。** `tools.profile` 设为 `minimal`，只暴露任务必需的工具。不做编码任务就没有随意的 shell，文件操作走受控接口而非裸命令。

**第三层：审批门。** 危险命令模式（`rm`、`dd`、`mkfs`、覆写配置等）进入人工审批队列。我日常保留这一层——它是唯一能拦住“语法合法但语义错误”命令的机制。

**第四层：快照兜底。** workspace 每次会话前做快照，配合 git。这层不阻止破坏，只保证破坏可回滚。

配置示例（字段名以你所用版本文档为准）：

```jsonc
"agents": { "defaults": { "sandbox": {
  "mode": "all", "scope": "agent",
  "docker": { "binds": ["~/ws:/workspace:rw"], "network": "none" }
}}},
"tools": { "profile": "minimal" }
```

## 踩坑点

1. **把 `$HOME` 挂进沙箱**。图省事挂整个 home，第一层防线直接塌掉。引用资料用只读挂载（`:ro`），写权限只给 workspace。
2. **宿主机侧的 MCP 工具绕过沙箱**。文件系统类 MCP server 跑在 gateway 进程里，天然摸得到宿主文件——沙箱只约束容器内行为。装第三方文件工具前先确认它的路径白名单能力，否则等于给 agent 开侧门。
3. **容器 root / privileged / 挂 docker socket**。这三件事任何一件都会让隔离形同虚设。
4. **workspace 内符号链接**。链向 `/etc` 的软链是模型最爱“顺手”跟随的东西，建 workspace 时清理干净。
5. **只防删不防改**。实际事故里误覆写比误删更常见，git 比快照更重要。

## 可复用建议

- 沙箱当边界用，别当建议用；权限默认拒绝，按需放开。
- 每个任务独立 workspace + 独立 scope，别让多个 agent 共享一个目录。
- 定期看 `exec` 审计日志，被拒绝的命令是最好的风险画像。
- 上线前在隔离环境用对抗性 prompt 演练：明确要求它“删掉工作区之外的东西”，看它被哪一层拦下。拦下即合格，没拦下就回去补配置。

## 总结

OpenClaw 不赌模型不出错，而是假设它一定会在某个下午生成一条错误的 `rm`。容器隔离划定物理边界，工具策略收窄攻击面，审批门拦住语义错误，快照保证可回滚——四层单独看都不完美，叠起来才构成“agent 不会误删文件”这个结论。安全不是一句提示词，是一组默认值。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/0b57b8a450973c49.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/740f3e3e6580a036.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/9fe2899c59ce7768.png)

