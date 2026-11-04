---
title: tools.md 实践：让 Agent 不再猜你的本地环境
feedId: 43485
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

用 OpenClaw 跑自动化一段时间后你会发现：同一份任务描述，在这台机器上一次通过，换一台就处处碰壁。原因多半不在任务本身，而在环境——Python 叫 `python3` 还是 `python`、包管理器是 brew 还是 apt、venv 在哪、代理怎么走、GPU 可不可用。这些信息在 agent 眼里默认是空的，它只能猜。

tools.md 就是为填这个空而生：放在 agent 工作区里、每次会话自动加载的 Markdown，把本机工具的调用方式和环境差异写成它可以直接查表的事实。

## 问题

不写 tools.md 时，这些知识散落在三处：系统提示词里的一两句、聊天记录里的临时纠正、你脑子里的肌肉记忆。后果是：

- agent 反复试错，先跑 `python` 失败再试 `python3`，浪费 token 也拖慢任务；
- 有些错误要等危险操作才暴露，比如清理脚本跑进了错误的环境；
- 换机器或重建容器后，之前纠正过的东西全部归零。

## 做法

**第一步：审计。** 别凭记忆写。让 agent 导出最近一周失败或重试过的命令，加上你的 shell history，列出高频踩雷点。通常 80% 的问题来自 5~6 个工具。

**第二步：写初版，命中率优先。** 行数不是目标，每条都应该是"实际踩过"的：

```markdown
# tools.md
## Python
- 一律用 /opt/venv/bin/python3，系统没有裸 python
- 装包前先 source /opt/venv/bin/activate

## 平台差异
- macOS 用 brew；Linux 容器用 apt，无 sudo
- 代理仅在 https_proxy 已设置时可用，勿自行 export

## 禁止
- /data/raw 只读，不要写入
- 不要全局 pip install
```

**第三步：挂载与分层。** 项目级 tools.md 放仓库根目录，随代码提交；机器级差异（绝对路径、本机端口）放全局工作区，项目级覆盖全局。如果框架只认 AGENTS.md，就在其中加一行指向 tools.md。原则：可复现的进仓库，绑机器的进全局，两边不重复。

## 踩坑点

- **写成散文。** 大段介绍 agent 不会当约束看，用短句、祈使句、列表，一行一条。
- **路径写死多处。** tools.md、MCP 配置、脚本里各存一份绝对路径，改一漏二。让 tools.md 当唯一事实源，其余引用环境变量或启动脚本。
- **贪多。** 超过百行后加载成本高于收益，稳定事实交给 `scripts/doctor.sh` 这类探针脚本，文件里只留结论。
- **过期比缺失更糟。** agent 会无条件相信它，环境一变就同步更新，拿不准的条目直接删。
- **混入敏感信息。** token、内网地址走环境变量，别写进文件。

## 可复用建议

- 让 agent 参与维护：会话结束前让它 diff 当次环境与 tools.md，提议增删，人来审核合并。
- 加一条 CI 校验：对文件里声明的每个命令跑 `command -v`，缺失即失败，防止悄悄过期。
- 团队里把它当 onboarding 文档用：新人和新 agent 需要的其实是同一份"本机怎么跑"。

## 总结

tools.md 的本质，是把环境知识从会话里的口头约定，变成版本化的工程事实。它不解决所有环境问题，但让 agent 从"猜"变成"查"——改动成本一行，收益当次会话就能看到。建议从你今天踩的最后一个坑写起。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/60b789a180f9a332.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/82fe950b089b5b4e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/79b0713b793596f6.png)

