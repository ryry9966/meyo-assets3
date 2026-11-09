---
title: OpenClaw 的 Sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 43943
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

跑过自动化 Agent 的人多少都遇到过这种时刻：让 Agent 清理临时目录，它把路径解析错，删掉了不该删的东西。OpenClaw 的定位是常驻的个人/团队自动化网关，Agent 手里有 exec、文件读写、MCP 工具一整套能力，误删文件不是假设性风险，而是必须正面处理的问题。

社区里常见的回答是“在 system prompt 里写清楚别乱删”。实践证明这不够：提示词是建议，不是边界。模型会犯错、会被注入、会把相对路径和通配符解析错。OpenClaw 的思路是把安全放进 sandbox 模型里，用结构保证，而不是靠模型自觉。

## 问题拆解

误删文件本质上是三个问题叠加：

1. **能力面过大**：Agent 的 shell/文件工具默认作用于整个宿主机文件系统；
2. **决策不可靠**：LLM 对路径、通配符、用户意图的理解存在概率性错误；
3. **无恢复手段**：真删了就是真删了，`rm` 不进回收站。

## OpenClaw 的分层做法

Sandbox 是几层叠加，缺一层由下一层兜底：

**第一层：运行时隔离。** 工具执行放在容器沙箱中（默认启用），Agent 能看到的只是挂载进去的 workspace，宿主机其余路径在沙箱内不存在。就算模型执行 `rm -rf /`，删的也只是沙箱里那份最小环境。

**第二层：路径策略。** workspace 按只读区 + 可写 scratch 挂载，所有写/删调用前做路径归一化：先 resolve 符号链接和 `..`，再判断目标是否落在允许根内。注意：简单的前缀字符串检查是不够的，symlink 逃逸是最常见的绕过手法。

**第三层：工具权限网关。** 高危操作不藏在自由 shell 里，而是独立建模成工具（如 `file.delete`、`file.move`），在配置中显式授权，未列入 allowlist 的能力默认拒绝。exec 命令经解析器识别破坏性模式，命中即拦截或转人工确认。

**第四层：删除进回收站。** 即使前面全部放行，`file.delete` 的默认行为也是把目标移入 workspace 内的 `.trash/` 隔离目录并保留元数据，到期才真正清理。误删从“事故”降级为“可撤销操作”。

示意配置（字段名以你所用版本文档为准）：

```yaml
sandbox:
  enabled: true
  mounts:
    - src: ./workspace
      dest: /workspace
      mode: rw
  tools:
    file.delete: trash   # 可选 confirm / deny
    exec:
      block_patterns: destructive
  trash:
    dir: /workspace/.trash
    retain_days: 7
```

## 踩坑点

- **图省事关沙箱，或把 `$HOME` 挂进 workspace**：等于四层防线拆掉三层，这是误删事故里最常见的原因。
- **symlink 逃逸**：workspace 里一个指向外部的软链，就能让前缀检查形同虚设。务必确认你的版本做了 resolve 后校验。
- **用正则黑名单拦 shell 命令**：`cd / && rm`、子 shell、变量展开都能绕过简单正则。要么用真正的解析器，要么收紧 exec 权限。
- **批准疲劳**：每个操作都弹确认，人很快会无脑点允许。应把确认收敛到少数真正高危的工具上，其余靠沙箱兜底。
- **提示词注入**：Agent 读到的网页、文档内容可能诱导它执行删除。这类威胁只能靠第一、二层的边界兜住，提示词约束只能算附加项。

## 可复用建议

- 一句话原则：**默认拒绝，按需放行；写操作限定 workspace，破坏性操作先进隔离区。**
- 给 Agent 的破坏性能力单独建工具，不要依赖“自由 shell + 提示词”。
- 保留工具调用的完整审计日志（工具名 + 参数 + 结果），出事能复盘，没事能改进策略。
- 定期做红队演练：故意给 Agent 一条模糊或危险的清理指令，验证拦截链路是否如预期生效。

## 总结

"Agent 不会误删文件”，不是因为模型足够聪明，而是因为错误的代价被结构限制住了：看不到宿主机文件系统、越界路径被拒、高危操作被网关拦下、真删了也先落在回收站。提示词影响的是概率，sandbox 决定的是下限。搭自动化时，先假设模型一定会犯错，再设计系统——这是 OpenClaw 这套安全模型最值得借鉴的地方。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/d0f1007fd9192ded.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/74fd4a297403727c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/3958590687ea49a7.png)

