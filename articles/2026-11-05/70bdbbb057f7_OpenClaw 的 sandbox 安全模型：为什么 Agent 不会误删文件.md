---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 43527
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

跑过带 shell 工具的 Agent 的人，多半做过同一个噩梦：模型看错路径，`rm -rf` 指向了不该指的地方。OpenClaw 的回答不是“把 prompt 写得更谨慎”，而是一套架构级的 sandbox 模型，核心假设只有一条：**模型输出不可全信，所以每一次工具调用的破坏半径必须被结构性限制**。

## 问题

先对齐“误删”到底是什么。它不是模型“想删文件”，而是它可能：

- 幻觉出路径，把 `/tmp/xxx` 当成 `~/xxx`；
- 在“帮我清理一下”这类模糊意图下，执行了带递归且无确认的命令组合。

这些是概率事件。prompt 约束（“你不许删文件”）只能降低概率，不能消除风险。真正的安全边界必须建在执行层，而不是语言层。

## OpenClaw 的做法

它把工具执行拆成几层，每层独立设防：

**1. 文件系统收窄。** 沙箱默认只把 workspace（`~/.openclaw/workspace`）挂载进容器。对 Agent 来说“世界”就是这个目录：`ls /`、`cat /etc/passwd` 都能跑，但看到的是容器内部，宿主机上未挂载的路径根本不可见。

**2. 执行环境隔离。** `sandbox.mode` 设为 `non-main`（主会话之外全部进沙箱）或 `all` 时，`exec` 的 shell 命令、文件工具的写入、甚至浏览器都发生在一次性 Docker 容器里。容器以非 root 用户运行，可关闭出站网络，生命周期与宿主机无关——删掉重建即恢复干净。

**3. 高危操作审批门。** 确实需要落到宿主机执行的 elevated exec 有审批机制：把 `rm -rf`、`sudo`、`curl | sh` 这类模式放进 deny/审批列表，没有明确确认就卡住。

**4. Gateway 边界。** Gateway 默认只监听回环地址，渠道侧用 pairing/allowlist。这一层不直接管文件，但保证“谁能驱动这个 Agent”也可控。

四层叠加，结论很简单：**一次误删的最大破坏半径 = 挂进沙箱的内容 + workspace 里未提交的改动**。

## 怎么验证

不必信文档，两分钟自测：

1. 开一个非主会话，让 Agent 执行 `ls ~`，与宿主机对比；
2. 让它在 workspace 里 `touch test && rm test`，检查宿主机目录；
3. 让它尝试 `rm -rf /tmp/whatever`，宿主机应毫无变化；
4. 删掉容器重建，确认状态不外泄。

## 踩坑点

- **图省事关沙箱。** `sandbox.mode: off` 意味着 exec 直接落宿主机，上面几层全部失效；
- **挂载贪多。** 把整个 home 目录挂进容器，等于把删除权交还模型；
- **容器里留了 sudo，或把 Docker socket 挂进去**，这些都是现成的逃生通道；
- **拿主会话做测试就下结论。** `non-main` 模式下主会话本身是 elevated 的，测出来不算数；
- **把 prompt 承诺当边界。** 模型说“我不会删”，只能当意图听，不能当保证用。

## 可复用建议

- 把 workspace 当耗材：初始化成 git 仓库，Agent 改动前先 commit，`git diff` / `restore` 就是你的 undo；
- 一个 Agent 一个 workspace 一个容器，最小权限；
- 审批列表定期维护，真正高危的模式写进 deny 而不是靠人工盯；
- 高风险任务先在独立 Agent 的空白 workspace 里彩排；
- 沙箱不替你做备份，“绝不能丢”的数据请自行兜底。

## 总结

OpenClaw 的安全模型没有黑魔法：少挂载、隔离执行、高危加门，把“模型说了什么”的责任从承诺层挪到结构层。Agent 不会误删文件，不是因为它足够聪明，而是因为它的手脚被框在了够不到你文件的地方。对挂载进去的那部分，用 git 和备份补上最后一环，就形成了完整闭环。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/5fbe05860d20a50c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/04e828b763be6be9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/b50f5a24b6e33c9a.png)

