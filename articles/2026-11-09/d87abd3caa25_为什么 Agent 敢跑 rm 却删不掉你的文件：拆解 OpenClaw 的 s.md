---
title: 为什么 Agent 敢跑 rm 却删不掉你的文件：拆解 OpenClaw 的 sandbox 安全模型
feedId: 43868
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

OpenClaw 的 Agent 默认带 exec 工具：模型可以真的执行 shell 命令、读写文件。能力越真实，问题越尖锐——模型"手滑"怎么办？很多人第一反应是在系统提示里加一句"不要删除文件"，但这本质上是把安全寄托在概率上。OpenClaw 的设计思路相反：不信任模型行为，用工程边界兜底。

## 问题

单靠 prompt 约束不可靠：长上下文遗忘、工具返回内容注入、路径拼接幻觉，都可能让模型"好心办坏事"。而如果 exec 直接跑在宿主机上，一次错误的相对路径展开就是一次真实的 `rm -rf`。核心问题不是"让模型更听话"，而是：**让 Agent 有手，但手够不到不该碰的地方。**

## 做法：四层防御

**1. 沙箱隔离（最外层，起决定作用）**

把 `agents.defaults.sandbox.mode` 设为 `all`（至少 `non-main`）后，exec 和文件工具都在 Docker 容器内执行。容器内看到的文件系统不是宿主机，`rm -rf /` 删掉的是容器自己的可写层。示意配置（OpenClaw 迭代快，字段名以当前版本文档为准）：

```json5
agents: {
  defaults: {
    sandbox: {
      mode: "all",
      scope: "agent",        // 排查插件时临时改 "session"
      workspaceAccess: "rw", // 仅 workspace 可写
      docker: {
        binds: ["/srv/repo:/repo:ro"],
        network: "none",
      },
    },
  },
},
```

**2. 挂载最小化**

workspace 以读写挂载，是 Agent 的"作业区"；其他宿主目录一律不挂，或只读挂。想让 Agent 读代码库？用 ro bind。要改？单独挂子目录，绝不要把 `$HOME` 整个挂进去。

**3. 命令策略（辅助层）**

exec approvals / 允许列表可以拦 `sudo`、`curl | sh` 这类高危模式。但注意它防不住 `bash -c "..."` 变体，只能当第二道闸，不能替代沙箱。

**4. 可恢复性（兜底层）**

workspace 放进 git 仓库 + 定时快照。沙箱拦"出格"操作，git 兜"合法但错误"的操作。

**5. 验证，而不是假设**

跑 `openclaw doctor` 确认 Docker 正常，然后故意让 Agent 执行 `ls /` 和一次无害写测试，确认它看到的确实是容器视图。把这一步写进新环境 onboarding。

## 踩坑点

- Docker daemon 没起时，部分环境会降级为宿主执行。配置写了 ≠ 生效，必须演练验证。
- `mode: non-main` 只覆盖子 Agent；主会话天天跑 exec 的场景请直接上 `all`。
- 把 `$HOME` 挂进容器 = 没有沙箱。
- `workspaceAccess` 配错时 Agent 连自己 workspace 都写不了，报错很迷惑。
- "容器缺依赖就关沙箱"是最常见的退化路径。正确做法是把依赖固化进镜像或初始化命令。
- 群聊里 approvals 弹窗被顺手批量确认，等于没有审批。

## 可复用建议

记住防御排序：**边界（沙箱）> 策略（只读挂载、命令过滤）> 可恢复（git/快照）> 提示词（只是 UX，不是安全手段）**。

- 任何新 MCP server 或插件，先在 `scope: session` 的临时容器里试跑。
- 只读需求一律 `ro` 挂载，需要写入再单独开子目录。
- 每次升级 OpenClaw 后重跑一次"误删演练"，确认分层仍然成立。

## 总结

OpenClaw 敢把 shell 交给模型，不是因为模型听话，而是因为"手"被限制在一个可丢弃的容器里：默认只有 workspace 可写、外部目录默认不可见、网络默认收紧，再加 git 兜底。误删在这个模型里从"事故"降级为"可恢复事件"。安全从来不来自模型自律，而来自默认值的组合。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/fd2840c594dd1133.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/759ad1b3b7dd5279.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/a73e5787a09666f6.png)

