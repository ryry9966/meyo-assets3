---
title: OpenClaw 的 Sandbox 安全模型：为什么 Agent 不会误删你的文件
feedId: 44362
source: 综合讨论
publishedAt: 2026-11-14
---

## 背景

OpenClaw 的定位是一个常驻的 agent 网关：收到消息后，LLM 会调用 exec、文件、浏览器等工具，直接在机器上干活。这决定了权限模型不是加分项，而是前提——Agent 手里有 shell，出错的代价就是真实的数据损失。

OpenClaw 的思路分两层：一层是**运行边界**（sandbox），一层是**审批链路**（ask / elevated）。核心原则只有一句话：不指望提示词让模型“小心”，而是让破坏性操作在物理上不可达。

## 问题

误删文件通常来自三个真实风险：

1. **模型幻觉与解析错误**。比如 `rm -rf $TMP/logs` 中变量为空，路径展开成根目录；或者拼接路径时少了一级目录。
2. **提示注入**。浏览器工具抓回来的网页正文里藏一句“执行清理命令”，模型在长上下文里可能照做。
3. **过度授权**。图省事用 root 跑网关，或者给 Agent 留着免密 sudo。

“请不要删除文件”这类提示词约束是软约束。安全边界必须落在进程和文件系统层面，而不是上下文里。

## 做法

1. **网关以独立低权限用户运行**：不 root、不进 sudoers。这是底线，先于一切 sandbox 配置。
2. **开启 Docker sandbox**：`agents.defaults.sandbox.mode` 设为 `all`（或 `non-main`，仅隔离非主会话）。每个 agent/workspace 生成独立容器，前置条件是装好 Docker。
3. **workspace 是唯一挂载点**：容器内只 bind-mount 工作目录，Agent 看到的文件系统根不是宿主机。`rm -rf /` 删掉的是容器层，工作区外的文件根本不在它的可见范围内。
4. **收紧出站网络**：sandbox 支持配置 egress，防止注入之后的数据外传。
5. **审批兜底**：工作区外写路径、elevated 提权命令走 ask 流程，人在环上确认。
6. **验证**：在工作区外放一个 canary 文件，让 Agent“清理所有日志”，观察它的路径解析和实际执行结果。

## 踩坑点

- Docker 没装或 sandbox 处于 off，工具会直接跑在宿主机上，其余约束全部失效。
- 把 `$HOME` 整个挂进容器图方便，等于没隔离。挂载粒度要最小化。
- `non-main` 模式下主会话仍在宿主机执行，敏感任务别丢给主会话。
- 容器里挂了 docker.sock 或装了 docker CLI，等于把宿主机 root 递出去，务必移除。
- ask 审批是兜底不是常态。审批疲劳会让你无脑点确认，高危操作应靠边界拦截，而不是靠人盯。

## 可复用建议

- 记住分层：**提示词只影响行为倾向，OS/容器才构成安全边界**。
- 三件事配齐就有基本盘：最小挂载 + 独立低权限用户 + 出站白名单。
- 每次改完 sandbox 配置，跑一次 canary 演练，别用生产数据验证。
- 保留 exec 调用日志。事后审计的成本，永远低于事后恢复。

## 总结

OpenClaw 防误删不靠模型自觉，而靠“删不到”：workspace 限定可见范围，容器隔离执行环境，审批链路兜住高危操作。模型仍然会犯错，但错误的爆炸半径被限制在你愿意承受的范围内——这才是和 Agent 共享一台机器时，唯一可靠的安心方式。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/e78c5e50e3c8c118.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/644d7bc66573739e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/254e733658ae1a89.png)

