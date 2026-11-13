---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 44305
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

用 OpenClaw 这类 agent 框架跑自动化的人，大概率遇到过这类场景：同一个任务，在公司电脑上一次跑通，换台机器就到处报错；或者 agent 反复尝试调用一个根本没装的 MCP 工具。根源往往不是模型能力，而是 agent 对"这台机器长什么样"一无所知。

OpenClaw 的工作区里本来就有 tools.md 这个约定文件（和 AGENTS.md 同属启动时读取的文档），但很多人要么留空，要么塞几行就再没维护过。这篇讲怎么把它用成管理本地配置和环境差异的主力，而不是摆设。

## 问题

常见做法是把环境信息塞进 system prompt，或者散落在 .env、shell alias、个人笔记里。三个具体后果：

1. **硬编码不可移植**。prompt 里写死 `/Users/xxx/...` 路径，换机器、换人就失效。
2. **agent 会猜**。没被告知某工具不可用时，它会反复重试，浪费 token 也浪费你的时间。
3. **事实无单一来源**。MCP 配置、依赖版本、端口占用散落各处，排障全靠回忆。

## 做法

核心思路：把 tools.md 当作环境事实的单一来源，关键是**分层**和**部分生成**。

**第一步：分两层建文件。**
- 全局层：`~/.openclaw/workspace/tools.md`，只放"跟人走"的事实——本机 OS、常用路径、装了哪些 MCP server、全局性的坑（比如代理端口）。
- 项目层：仓库根目录 tools.md，只放"跟项目走"的事实——依赖的工具、约定的脚本入口、环境变量名（不是值）。
- 优先级写进文件开头：项目层覆盖全局层。

**第二步：机器相关的内容别手写，用脚本生成。**
十几行脚本采集 node/python 版本、Docker 状态、MCP server 存活情况，追加到 tools.md 的 generated 段，带时间戳。开工前跑一次，可以挂进 Makefile 或 pre-task hook。

**第三步：给个最小模板。**

```markdown
# tools.md
## 覆盖规则
本项目文件优先于全局文件。
## 可用工具
- playwright-mcp: 可用
- 数据库: 本地 postgres, 端口 5432
## 约定
- 一律用 make task-x，别直接调 python
## 禁止
- 不要动 ./data 下的文件
<!-- generated: 2025-01-15, docker: up, node: v20.11 -->
```

**第四步：验证加载链路。**
会话里问一句"列出你当前可用的工具"，确认 agent 真的读了它。别默认生效。

## 踩坑点

- **别放密钥**。tools.md 会进上下文，secret 一律只写"去 .env 找 KEY_NAME"。
- **别让它膨胀**。见过涨到 400 行的，一半是过期信息。规则：不超过一屏，过期就删，历史交给 git。
- **两层重复内容必然打架**。分工清楚：全局只写机器事实，项目只写项目事实。
- **路径差异是高频事故源**。Windows/Unix 路径统一让脚本生成，别手敲。
- **generated 段必须带时间戳**，否则你分不清"三年前的 docker 状态"和现状。

## 可复用建议

- 把 tools.md 当代码管理：项目层进版本库，全局层进 dotfiles。
- 写个 `agent doctor` 脚本：校验 tools.md 声明的工具是否真实可用，输出差异，50 行以内。
- 团队场景下，tools.md 的变更和代码变更一起过 review，环境变更同等重要。
- 排障顺序固定：先跑 doctor，再看 generated 时间戳，最后才怀疑 agent。

## 总结

tools.md 不是新协议，只是把"环境事实"从 prompt 和记忆里挪出来，变成分层、部分生成、可校验的静态文件。价值在于：agent 少猜、你少解释、换机器少踩坑。成本是一个 md 加一个小脚本，收益是把环境差异从隐性知识变成显性资产。建议从当前项目建最小模板开始，跑两周再迭代。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/ceb2aad28819c314.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/d11c0e327860760b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/ec9eac39973d4c13.png)

