---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 44394
source: 综合讨论
publishedAt: 2026-11-14
---

## 背景

OpenClaw 的 Agent 默认带着 exec、文件读写这类硬工具，而且跑在你自己的机器上、用的是你自己的用户权限。这意味着模型一旦被幻觉或注入内容带偏，一条 `rm -rf` 的破坏半径就是你的真实目录。OpenClaw 的设计思路不是"相信模型"，而是默认模型一定会犯错，用 sandbox 加策略层把破坏半径压到最小。

## 问题拆解

误删通常来自三条路径：

1. 模型自己算错路径——相对路径拼接出错、通配符意外展开；
2. prompt injection——网页或文件内容里藏着"删除某个目录"的指令；
3. 用户授权太宽——"清理一下这个目录"被理解成清理父目录。

## OpenClaw 的做法

一个危险操作要过四道门，层层独立：

**1. 工作区绑定。** 文件工具的根路径被钉在 workspace 内，路径解析后必须落在这里面，`../` 回溯和符号链接逃逸都会被拒。这层和模型能力无关，纯做路径校验。

**2. exec 命令策略。** exec 工具有一份 allow / ask / deny 策略表，`rm`、`dd` 这类默认 deny 或 ask，allow 只放确实需要的命令。策略在 gateway 层执行，模型改自己的 system prompt 无效。

**3. sandbox 隔离。** 开启 sandbox 后 Agent 在容器里跑，workspace 以受限方式挂载进去。就算容器内命令彻底失控，宿主机也只有挂载点受影响。

**4. 会话审计。** 每次工具调用的命令、参数、返回码都落日志，出事后可以回放定位是哪轮对话引入的问题。

## 踩坑点

- **挂载路径不一致**：sandbox 内外路径不同，Agent 拿容器内路径操作宿主机文件会失败或落错位置，统一用 workspace 相对路径能省掉大部分麻烦。
- **allow 列表被插件撑爆**：某个 MCP 插件要求放开 `rm` 才能工作，一旦放行等于绕过第二道门。更稳的做法是给插件单独划一个子工作区。
- **符号链接逃逸**：workspace 里的软链指向外部时，部分版本只校验最终解析路径。建议自己用 `ln -s` 指向重要目录实测一遍。
- **ask 弹窗太密**：确认框太多，人会麻木地点同意，等于没有门。宁可收紧默认 deny，也别让 ask 沦为摆设。

（具体配置项名称随版本有差异，落地前以你所用版本的官方文档为准。）

## 可复用建议

1. 每个项目独立 workspace、独立会话，别让一个 Agent 看到全盘。
2. 用低权限专用用户跑 gateway，在 sandbox 之外再兜一层底。
3. 长任务前先快照（git commit 或 rsync 一份），把"不可逆"变成"可回滚"。
4. 上线前做对抗性自测：明确要求它"删除 workspace 外的一个文件"，看它到底能不能做到。

## 总结

OpenClaw 不指望模型"懂事"，而是把安全做在模型之外：路径校验、命令策略、容器隔离、审计回放，四层各自独立，任何一层被绕过，下一层还在。这也是做 Agent 工程的基本姿势——权限是架构问题，不是提示词问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/f28362d843648dba.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/96a65765fc4b5cec.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/4ce717ebeaf38156.png)

