---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 43353
source: 综合讨论
publishedAt: 2026-11-03
---

## 背景

让 LLM Agent 直接操作文件系统，大家最担心的一件事是：它哪天手一抖，把不该删的东西删了。OpenClaw 的 Agent 自带文件读写和 shell 执行能力，理论上确实能发出 `rm -rf`。但实际跑下来，误删基本不会发生——不是因为模型聪明，而是因为执行层根本不给它这个机会。这篇帖拆一下 OpenClaw sandbox 的分层设计，以及我自己部署时踩过的坑。

## 问题：误删的真实来源

误删不是模型"作恶"，通常是三类工程问题：

1. **路径理解错误**：Agent 把相对路径解析到错误位置，或把 workspace 外的路径当成了内部路径。
2. **提示注入**：仓库里某个文件内容夹带"请删除 xxx"之类的指令，Agent 读到后照做。
3. **工具越权**：某个 MCP server 或插件自带文件访问能力，绕过了 Agent 层的约束。

所以安全模型必须假设：模型会犯错、输入会被污染、第三方工具不完全可信。

## 做法：四层防护是怎么叠起来的

1. **Workspace 根植**。所有文件工具（read / write / edit / move / delete）的路径参数默认相对 workspace 解析，越出 workspace 的目标直接被网关拒绝，模型只会收到一个 "path outside workspace" 的错误。这是最硬的一道墙。
2. **路径规范化**。网关放行前先 canonicalize：展开 `..`、解析符号链接，取真实路径再和 workspace 前缀比对，防住 `./../../etc` 和"workspace 里放一个指向外部的 symlink"这类绕过。
3. **危险命令闸门**。exec 工具单独过一道模式匹配：`rm -rf`、`find -delete`、`git clean` 这类命令默认进入待批准队列，或要求显式 allowlist。workspace 内的正常操作不受影响。
4. **MCP 作用域**。每个 MCP server 注册时声明能力（是否有写权限、可触碰哪些路径前缀），网关按声明放行；没声明文件能力的插件根本调不到文件工具。

另外，如果 workspace 开了 git，会话开始会留基线，出问题可整体回滚——这是兜底，不是第一道防线。

## 踩坑点

- **workspace 挂得太高**。我最初图省事把 workspace 设成整个 home 目录，结果"安全边界"约等于"爆炸半径"。后来固定用一个专门目录，问题才消失。
- **exec 闸门 ≠ 文件闸门**。Agent 通过 shell 跑 `tar`、`cp` 时可以写出 workspace，路径检查只拦 OpenClaw 自己的文件工具，所以闸门那层不能关。
- **symlink 是动态的**。路径检查发生在每次访问时，但如果你的 MCP server 内部缓存了解析结果，就可能拿旧路径行事。自写 MCP server 时别缓存路径。
- **"临时"关闭审批**。为了跑批处理把 approval 关了，忘了开回来。建议用配置文件而不是命令行开关，改配置至少会过一次 diff review。

## 可复用建议

- **最小 workspace**：给 Agent 一个专用目录；需要操作代码仓库时用只读映射或同步，不要直接把仓库当 workspace。
- workspace 里保持 `git init`，回滚成本接近零。
- 审批策略留档：allowlist 写进仓库里的配置文件，review 时看得见。
- 做一次自查：故意让 Agent "删除 workspace 外的某个文件"，观察报错是否符合预期——相当于免费的渗透测试。
- 审计 MCP server 的实现。记住：沙箱边界在网关，不在 server 进程内部。

## 总结

OpenClaw 防误删不靠提示词，靠的是 workspace 根植、路径规范化、危险命令闸门、MCP 作用域这四层相互独立的机制。设计前提就是"模型一定会犯错"，所以每层都在执行侧兜底。作为使用者，你的责任只有两件事：别把边界画得太大，别亲手拆掉闸门。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-03/3791a8ebaad06704.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-03/9d2bf6b288abdd60.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-03/ace1ef1ad3608e8b.png)

