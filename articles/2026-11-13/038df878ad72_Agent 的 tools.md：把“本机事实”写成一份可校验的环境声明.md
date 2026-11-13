---
title: Agent 的 tools.md：把“本机事实”写成一份可校验的环境声明
feedId: 44289
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

OpenClaw Agent 的能力来自三层：system prompt、内置工具、MCP/插件。但“这台机器上到底有什么”，Agent 其实靠猜，除非你告诉它。OpenClaw 有个工作区约定：在 agent workspace 根目录放一个 `TOOLS.md`（大小写按精确文件名匹配），会话启动时会作为补充说明读入上下文。用好了它是环境声明文件，用不好就是第二份没人维护的 README。

## 问题

实际用下来，环境差异导致的翻车基本三类：

1. **幻觉工具**：Agent 假设机器上有 `ffmpeg`、`jq`，实际没装，反复报错后才开始“探索”。
2. **路径与平台差异**：macOS Homebrew 的 bin 在 `/opt/homebrew/bin`，Linux 在 `/usr/bin`；同一份 workspace 在笔记本、台式机、VPS 上表现不一致。
3. **约定散落**：某个 MCP server 怎么启动、项目脚本入口是哪个，全靠翻聊天记录。

根因只有一个：Agent 上下文里缺少一份“本机事实清单”。

## 做法

**第一步：固定骨架。** 按统一分节写，别自由发挥：

```markdown
# 本机环境
- macOS 15 / Apple Silicon，brew bin 在 /opt/homebrew/bin
# 可用 CLI
- ffmpeg 7.1（转码）；rclone 1.7（网盘同步）
# MCP servers
- filesystem：npx 启动，根目录限定 ~/projects
# 禁区
- 别动 /etc/hosts；不要跑交互式命令
```

**第二步：脚本生成初稿，人工审阅。** 写一个二十行的 dump 脚本，遍历你关心的 CLI 列表跑 `command -v` + `--version`，输出 markdown 草稿。事实由机器采集，取舍由人决定。

**第三步：区分“公共约定”与“本机事实”。** workspace 用 git 同步多机时，公共部分（脚本入口、MCP 约定）进仓库；机器相关部分拆到 `tools.local.md` 并 gitignore，在公共文件里留一行指引。别把两台机器的事实硬塞进同一份文件。

**第四步：给变更设触发点。** 装新工具、改 MCP 配置、大版本升级后，重跑 dump 脚本 diff 一次。也可以挂 cron 每周对比“文档声称可用”和 `command -v` 实测结果，漂移就报警。

## 踩坑点

- **写成第二份 README**：超过两屏重点就被稀释，还会灌水上下文。tools.md 只回答“有什么、怎么用、别碰什么”。
- **写“应该有”而非“实际有”**：过期声明比没有更糟，Agent 会理直气壮地按错误路径执行。
- **塞敏感信息**：只写环境变量名，token 永远不进 tools.md。
- **位置或大小写不对**：文件必须在工作区根目录、文件名精确匹配，否则静默不生效；改完新开会话验证一次。
- **职责混淆**：行为规范、人格设定留给 AGENTS.md / SOUL.md，tools.md 不掺和。

## 可复用建议

- 把 dump 脚本做成模板，新机器初始化跑一遍，五分钟出 tools.md 初稿。
- “声明 vs 实测”的校验逻辑抽成独立小工具，本地和 cron 都能跑。
- 团队场景下它就是最好的环境 onboarding 文档：新人让 Agent 读它，比自己翻 wiki 快。

## 总结

tools.md 的本质，是把环境的隐式知识变成工作区的显式声明：内容要短、要真、要跟机器走。事实交给脚本采集，判断留给人，Agent 只负责照着执行——这才是管住环境差异的省力姿势。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/1a74619426bcc901.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/ba5a3ed2569f66b9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/3aa0fc6086b016c2.png)

