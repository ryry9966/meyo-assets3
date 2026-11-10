---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 43995
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景

跑 OpenClaw / Agent 工作流的人，大多不止一台机器：笔记本是 macOS，家里一台 Linux 小主机，可能还有 Windows。AGENTS.md、提示词、skills 可以随 dotfiles 同步，但真正执行命令的环境不一样——Python 有没有 `python` 别名、包管理器是 brew 还是 apt、有没有代理端口，每台机器都不同。模型读到的“世界”是同一份，脚下踩的机器却不是同一台。

## 问题

不显式管理这些差异，后果很典型：

- 提示词里写死 `/Users/xxx/projects/...`，换台机器就失效，agent 开始猜路径；
- agent 在只有 `python3` 的机器上反复试 `python`，浪费轮次，还可能装错环境；
- 工具升级后文档没跟上，agent 按旧命令执行，报错原因很难从日志里看出来；
- 更糟的隐患：有人为了“帮 agent 省事”，直接把 token 写进提示词。

## 做法

核心思路：在 workspace 放一份 `tools.md`，作为本机环境的唯一事实来源，并约定 agent 执行命令前先读它。

```markdown
# tools.md — 本机环境（agent 必读）

## 运行时
- python: 只有 python3（3.12），没有 python 别名
- node: v20，走 nvm，需先 source ~/.nvm/nvm.sh

## 路径
- 工作目录: ~/projects/openclaw；日志写 ./logs，不要写 /tmp

## 禁区
- 禁止全局 pip install，一律用项目内 .venv
- 禁止 sudo
```

然后做分层，解决“通用 vs 机器私有”：

1. `tools.md` 提交到 git，只放通用约定（禁区、目录规范）；
2. `tools.local.md` 加入 .gitignore，放绝对路径、GPU 状况、代理端口这类机器私有事实；
3. 在 AGENTS.md 里写明优先级：两者冲突时以 local 为准，与提示词冲突时以 tools.md 为准；
4. 收尾验证：换一台机器冷启动，让 agent 跑一次 smoke test——报 Python 版本、跑一遍测试、确认日志目录可写。

## 踩坑点

- **别写密钥值**。只写“已存在于 shell 环境变量 `XXX_KEY`"，值永远不落盘。
- **别和 MCP 工具清单重复**。MCP 注册的 tools 是动态的；tools.md 记录的是本机可执行环境（解释器、路径、包管理），两者是不同层面。
- **控制长度**。几十行以内，硬约束（禁止 sudo、禁止全局安装）放最前面，太长 agent 会跳读。
- **文档会腐烂**。升级依赖时顺手更新；也可以加一条周期任务，让 agent 自检 tools.md 里的路径是否还存在，失效就提醒你。
- **跨平台差异写进去比现场试错便宜**。Windows 的路径分隔符、引号转义，一行说明能省掉一串失败重试。

## 可复用建议

- 把 tools.md 当“机器能力清单”而非教程，一行为一条事实，能用表格就用表格（工具 / 路径 / 版本 / 备注）。
- 团队场景：仓库里放 `tools.example.md` 作模板，每人维护自己的 local 版本——模板进版本库，私有文件不进。
- 新环境接入流程固定为：先按模板填 local，再跑 smoke test，最后才交给 agent 干活。

## 总结

tools.md 做的事很简单：把“环境事实”从模型的猜测变成显式上下文。成本是一次几十行的编写，收益是更少的试错轮次、更少的幻觉路径，以及一份可以随机器迁移的配置。不用追求一步到位，先把你最常报错的那条命令写进去，就够了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/8e97cd52c44901ae.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/0a6e163ce7205a47.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/0d3466c21f0694c1.png)

