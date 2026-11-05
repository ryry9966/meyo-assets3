---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 43502
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

OpenClaw 的 workspace 里有一组约定文件：AGENTS.md 管行为规则，SOUL.md 管人设，而 TOOLS.md 的定位常被忽略——它是默认注入上下文、专门描述“环境事实”的文件。MCP 和插件解决的是能力接入，但 agent 真正动手时缺的是另一类信息：这台机器的包管理器是 brew 还是 apt、项目放在哪、内网哪些服务能直接访问。这些信息不该靠 agent 每次试探。

## 问题

常见的几种翻车：

1. 上下文里没有环境事实，agent 在 macOS 上调 `apt`，或者用 `pip` 全局装包，全靠猜；
2. 同一份 workspace 通过 git 同步到台式机、VPS、NAS，agent 在服务器上执行了只适合笔记本的路径；
3. 有人把规则、路径、注意事项全塞进 AGENTS.md 或系统提示词，上下文膨胀，行为和环境混在一起，跨机还互相打架；
4. 更危险的：直接把 API key 明文写进去。

## 做法

我给自己的 TOOLS.md 定了三条原则，再按固定骨架写。

**原则一：只写事实和指针，不写规则。** 行为约束归 AGENTS.md，TOOLS.md 只回答“这台机器有什么、在哪、怎么用”。

**原则二：一台机器一份。** 每个设备的 workspace 独立，公共部分（比如脚本仓库地址）走 git，机器差异留在本地，不进公共分支。

**原则三：密钥只给位置，不给值。** 写“token 在 `~/.config/xx/creds`，用前再读”，而不是把值贴上去。

骨架大致是：

```markdown
## 本机
hostname 检测命令、OS、架构、包管理器
## 工具清单
实际装的 CLI 及版本；没装什么（同样重要）
## 路径约定
项目根目录、venv 位置、compose 文件在哪
## 内网与远程
SSH alias、常访问的 LAN 服务
## 已知坑
代理、Python 版本别名、shell 差异
```

“没装什么”值得强调：明确写“本机没有 docker，容器任务转发到远端主机”，比让 agent 试错三轮省得多。

“本机”和“工具清单”两节可以由 setup 脚本自动生成，装完新工具跑一下，减少文档漂移。

## 踩坑点

- **写太长。** TOOLS.md 每轮都占上下文，超过一屏就该砍，目标是 3 分钟读完。
- **硬编码绝对路径。** `/home/alice/...` 换用户或进容器就失效，用 `~`，或让 agent 先 `echo $HOME`。
- **职责串门。** 规则写进 TOOLS.md，AGENTS.md 里又写一遍环境，两处迟早不一致，agent 会挑对自己有利的那个。
- **多机共用一份靠注释区分。** `<!-- 仅服务器 -->` 这种约定非常脆弱，不如直接按机分文件。
- **只写不更新。** 配合 heartbeat 或在 setup 脚本里加一个环境快照 diff 提醒，装了新东西就刷新。

## 可复用建议

- 把 TOOLS.md 当 runbook，不是说明书：短、事实性、可验证、带检测命令。
- 提供 hostname 检测命令，让 agent 能自己确认“我现在在哪台机器”。
- 密钥一律指针化。TOOLS.md 会进上下文，还可能被同步和备份，默认它不是安全存储。
- 定期通读一遍，删掉失效条目——这类文档的腐烂速度比你想象的快。
- 这套思路同样适用于 CLAUDE.md 等同类约定文件，迁移成本很低。

## 总结

TOOLS.md 解决的是 agent 自动化里最朴素的一环：让它少猜。把环境事实结构化地交给它，MCP 和插件的能力才能真正落对地方。一台机器一份、只写事实、密钥指针化——这三条做到，跨机翻车率会明显下降。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/85a7ff55e87c3aef.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/4bd4a7d77c66ef67.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/9599944ebcd1e172.png)

