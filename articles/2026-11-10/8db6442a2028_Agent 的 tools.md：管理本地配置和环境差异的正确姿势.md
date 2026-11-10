---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 44004
source: 综合讨论
publishedAt: 2026-11-10
---

# 背景

跑 OpenClaw 一段时间后你会发现，Agent 的“可靠程度”有一半取决于它对本地环境的了解。家里 Mac mini 上的 Node 是 nvm 管的，VPS 上是 apt 装的；一台机器有 Docker，另一台没有；笔记本走代理，服务器直连。这些差异不会写在任何官方文档里，Agent 只能猜。

猜错的表现很一致：`pnpm` 不存在就回退 npm、Python 用了系统版本而不是 venv、browser 工具打不开因为 profile 路径不对。每次都要在对话里口头解释一遍，说完就忘，下次换个会话再来一遍。

# tools.md 解决什么问题

OpenClaw 约定 workspace 根目录的 tools.md 会作为上下文注入 Agent。它和 AGENTS.md 的分工要划清楚：

- **AGENTS.md**：行为约定——怎么回复、什么该做、什么不该做；
- **tools.md**：环境事实——这台机器上命令怎么跑、路径在哪、有什么坑。

一句话总结：AGENTS.md 管“人格”，tools.md 管“地理”。

# 具体做法

1. **建文件**。workspace 根目录新建 tools.md，目标控制在 80 行以内。
2. **按固定骨架写**：机器概况（OS / 架构 / 资源）、运行时（Node / Python 版本与版本管理器）、命令偏好（pnpm 优先、Python 一律 `uv run`）、网络（代理地址、镜像源）、MCP 与插件注意事项（browser 用哪个 profile）、禁区（不要动的目录）。
3. **只写事实，不写教程**。“Node 由 nvm 管理，用 `nvm use 22`”就够了，不要展开讲 nvm 是什么。
4. **多机方案**：git 仓库里放一份 `tools.common.md` 存共性，每台机器维护自己的增量段；或者干脆每台机器独立一份、模板复制。不要用一个文件硬同步所有机器——机器差异本身就是内容。
5. **让它自愈**：Agent 因为环境问题失败一次，收尾时就让它把结论追加进 tools.md 的“已踩坑”小节，你审一遍再合并。

# 踩坑点

- **别写密钥**。tools.md 会被注入上下文，还可能随 git 同步。写“API key 在 `~/.config/xxx/env`，已 source”这种引用即可。
- **别贪长**。两百行的 tools.md 会稀释 Agent 的注意力，关键信息反而丢。80 行内，超了就拆。
- **别和 AGENTS.md 混**。在 AGENTS.md 里写“本机用 pnpm”，换台机器全错。
- **过期信息比没信息更糟**。升级了 Node、换了代理端口，记得同步改。每次排障后顺手核对一遍相关条目。

# 可复用建议

- 骨架固定、内容按机器填，新机器部署时 10 分钟能写完。
- “已踩坑”小节是整个文件价值最高的部分，鼓励它持续增长，但每条不超过两行。
- 每季度花五分钟通读一次，删掉不再成立的内容。
- 判断标准：如果某条环境事实开始影响行为决策（比如“不要自动 git push”），它属于 AGENTS.md，迁移过去。

# 总结

tools.md 的本质，是把“你对机器的了解”固化成 Agent 可读的上下文，让环境差异从对话里的口头解释变成仓库里的静态事实。写得克制、写得准确、及时维护，Agent 在任何一台机器上的行为都会稳定得多。它不是配置系统，只是给 Agent 的一页纸备忘——保持这个定位，就不会写歪。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/081b62ab6b5de2d0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/8047bee1168bbde1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/4f6007f4fe864df3.png)

