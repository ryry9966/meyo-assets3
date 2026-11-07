---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 43734
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

OpenClaw 的 Agent 能读文件、写文件、跑 shell，这是它有用的原因，也是它危险的来源。只要一条指令生成出错——比如 `rm -rf "$DIR/"` 里 `$DIR` 恰好为空——后果就是真实数据丢失。很多人以为"不误删"靠的是模型够聪明，或者 system prompt 里写了"请谨慎操作"。这篇文章讲清楚 OpenClaw 实际的做法：**分层防御，默认拒绝**，而不是祈祷模型懂事。

## 问题：事故的真正成因

让 Agent"清理项目临时文件"，它可能：把相对路径当绝对路径展开、通配符覆盖过宽、变量为空导致路径退化到根、甚至被注入内容（邮件正文、网页文本）操纵执行非预期命令。这类错误和模型能力关系不大——能力越强，能生成的破坏性指令反而越多。所以正确的问题不是"模型会不会犯错"，而是"**犯错时它能伤到哪**"。

## 做法：四层防线

1. **默认非全权**。写文件、执行命令这类工具不是开箱即得的；full access 是显式开启的选项，不是默认状态。
2. **Docker 沙箱**。`sandbox.mode` 设为 `non-main` 或 `all` 后，Agent 的 shell 跑在容器里：workspace 以可写卷挂载，宿主其余文件系统根本不可见。看不见的文件删不掉——这比"不许删"强得多。
3. **工具策略与会话分级**。tool policy 的 allow/deny 列表决定哪些工具可用；定时任务、群聊触发的非主会话默认走沙箱，只有主会话可以按需提权。
4. **人工审批**。非全权模式下 exec 会触发 approval，最终决定权留给人。

示意配置（字段以你当前版本的 docs 为准）：

```yaml
agents:
  defaults:
    sandbox:
      mode: non-main   # off / non-main / all
```

## 验证步骤

配置写了不等于生效，建议做一次破坏性演练：

```bash
# 让 Agent 在沙箱内执行，逐条确认结果
ls /            # 应看不到宿主文件系统
rm ~/notes.md   # 应失败，路径不存在
touch /tmp/x    # 容器内成功，宿主无痕
```

再确认 workspace 内读写正常。全程几分钟，换一个明确的边界认知。

## 踩坑点

- **workspace 直接挂 `$HOME`**：等于把整个家目录交给 Agent，沙箱形同虚设。
- **全权模式 + 免密 sudo**：所有防线一次穿透。
- **为了装依赖放开全量网络**：egress 应收敛到镜像源或代理。
- **把 docker.sock 挂进沙箱**：容器拿到宿主控制权，典型的逃逸路径。
- **workspace 里的 symlink 指向宿主路径**：写操作可能顺着链接穿出边界。
- **只信 prompt 约束**：prompt 可以被注入内容覆盖，它不算防线，只是提示。

## 可复用建议

- **最小可写原则**：只有 workspace 可写，其余路径只读或不可见。
- **自动化任务一律沙箱**：cron、群聊触发、无人值守场景用 `non-main`，交互主会话才考虑提权。
- **权限配置纳入版本管理**：升级 OpenClaw 后重跑一次演练脚本，防止配置漂移。
- **备份仍是底线**：沙箱降低事故概率，但不消灭后果。

## 总结

OpenClaw 防"误删"的思路，不是让模型更可靠，而是让错误的爆炸半径收敛在 workspace 里：默认拒绝、边界隔离、人工兜底。模型会犯错是设计前提，不是意外。一个简单的自查标准：如果某类事故你承受不起，就别让 Agent 对相应路径有可见性——可见性本身就是攻击面。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/2d5b8896a4a4e0ec.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/5228091e3fb19f19.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/8cfacfc5a81a3c4a.png)

