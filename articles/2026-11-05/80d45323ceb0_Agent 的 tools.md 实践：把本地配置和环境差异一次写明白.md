---
title: Agent 的 tools.md 实践：把本地配置和环境差异一次写明白
feedId: 43544
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

OpenClaw 工作区有一组约定文件：AGENTS.md 管行为规则，USER.md 管个人偏好，MEMORY.md 管长期记忆。而 TOOLS.md 的定位常被误解——它不是工具说明书，而是 Agent 每次开工前“了解这台机器”的事实清单。当你在家用台式机、公司笔记本和一台 VPS 上各跑一个实例，或者同一台机器上多个 agent 协作时，环境差异就是自动化翻车最常见的一类根因，这份文件正是收口点。

## 问题

没有 tools.md 时，Agent 对环境的认知来源只剩两类：试探和猜。典型翻车现场：

- 在 macOS 上想当然地 `apt install`；
- 项目明明在 `~/work/xxx`，它跑去 `~/projects` 找；
- 本机 8188 端口跑着 ComfyUI，它另起一个然后报端口占用；
- 最隐蔽的是记忆里的旧事实（比如已迁移的 venv 路径）和新现实打架，行为时对时错。

共性很明显：环境事实没有唯一权威来源，散落在提示词、会话历史和记忆里，互相污染。

## 做法

把 tools.md 当作“机器契约”来维护，几条实操：

1. **分层。** 全局模板放 dotfiles 仓库（所有机器通用的约定），每台机器再维护一份本地覆盖（具体路径、端口、硬件、禁区）。Agent 优先读工作区本地这份。
2. **只写事实，不写过程。** 一行一条，短句或表格。骨架示例：

```markdown
## Host: dev-laptop (macOS 14, Apple Silicon)
- shell: zsh；包管理: brew（无 apt）
- python: 3.12 @ ~/work/.venv/bin/python（uv 管理）
- 常驻服务: Ollama :11434；ComfyUI :8188（仅本机）
- 禁区: 不改 /etc/hosts；Docker 无 GPU
```

3. **控制体积。** 这个文件每次会话都会进上下文，我给自己的预算是 60 行以内。跑题的 runbook 一律回项目 README。
4. **和 AGENTS.md 联动。** 加一条策略：“发现实际环境与 tools.md 不符时，先停下来指出，不要按记忆硬做。”这一条比文件本身更能止损。
5. **更新有仪式感。** 换机器、升级系统、新增服务，当天就改 tools.md，待遇等同改 `.env.example`。可以写个十行的脚本 dump 版本号和关键路径，人工 diff 后粘进去。

## 踩坑点

- **放密钥。** 最多写引用（“见密码管理器某条目”或环境变量名），绝不写值——这份文件会整份进上下文和日志。
- **写成小作文。** 段落式描述 Agent 提取不可靠，还费 token。
- **和 USER.md、记忆重复。** 同一事实两处维护必然漂移，Agent 会随机采信其一。分工定死：机器事实只归 tools.md。
- **烂在仓库里。** 过期条目比没有条目更危险，它会被当作权威执行。给每节标个 `verified: 2025-06` 之类的日期，便宜但有效。
- **想当然它会被读。** 换工作区或新装实例后验证一次：直接问 Agent “这台机器的 python 在哪”，看回答是否引用了文件内容。

## 可复用建议

- 三个文件的分工口诀：AGENTS.md 管“怎么做事”，USER.md 管“为谁做事”，TOOLS.md 管“在哪做事”。
- 把 tools.md 纳入 git，机器级差异用独立的 host 段落管理，换机迁移时同步一份。
- 每季度或大版本升级后通读一遍，删掉三个月没被用到、也没被验证过的行。

## 总结

tools.md 不是什么高级机制，本质是把“运维交接文档”搬进了 Agent 工作区。但环境差异是自动化最常翻车的地方，一份小而准的事实清单，配合“不符即上报”的策略，能消掉其中大部分问题。成本是几分钟维护，收益是 Agent 少猜、少错、少重建环境。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/2769e5012cd5b324.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/f80efd59139b33c1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/e4b124e45427dd67.png)

