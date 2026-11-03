---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 43379
source: 综合讨论
publishedAt: 2026-11-04
---

# 背景

给 Agent 接上 shell 之后，最让人睡不着觉的问题只有一个：它哪天会不会手一抖，把 `~/projects` 给 `rm -rf` 了。LLM 不需要恶意才能删文件——一次路径幻觉、一句被误解的"帮我清理下旧文件"就够。OpenClaw 的回答不是"祈祷模型别犯错"，而是把删除能力从架构上隔离掉。

# 问题：误删不是概率问题，是权限问题

模型出错是常态：相对路径解析错、把"归档"理解成"删除"、在错误的 cwd 里执行通配符。传统做法是在 prompt 里写"请小心点"，这在工程上等于没做。OpenClaw 的思路是：**假设模型一定会犯错，然后让犯错的爆炸半径为零。**

# 做法：四层防线

**第一层：容器沙箱。** `sandbox.mode: "all"` 时，Agent 的 exec/read/write 全部发生在一个 Docker 容器里。宿主机文件系统根本不挂进容器——Agent 看到的"全世界"只有 workspace 那一个 bind mount。它不是"不敢删"，是物理上够不着。

**第二层：工具策略。** 即使在沙箱内，tool policy 还能做命令黑名单，把 `rm`、`dd`、`mkfs` 这类模式 deny 掉，exec 命令在进 shell 前先过一遍策略匹配。

**第三层：exec 审批。** 对没被策略覆盖的敏感操作，可以开审批，人工确认后才执行，形成"平时放行、危险动作拦一道"的节奏。

**第四层：Gateway 职责分离。** Gateway 进程跑在宿主机上，只做编排和消息路由，不替 Agent 在宿主机执行命令。Agent 的"手"被焊死在容器里。

配置示意（`openclaw.json`）：

```json5
agents: {
  defaults: {
    sandbox: { mode: "all" },
    tools: { deny: ["rm", "dd", "mkfs"] }
  }
}
```

# 踩坑点

1. **mode 选了 `"non-main"`**：默认主会话（你直接私聊的那个）不走沙箱，只有派生子 agent 走。很多人配完以为全沙箱了，其实主会话还能碰宿主机。要么显式 `"all"`，要么明确主会话的用途。
2. **bind mount 挂太宽**：为了"方便"把整个 HOME 挂进去，沙箱就名存实亡。挂什么，Agent 的世界就只有什么。
3. **把 `docker.sock` 挂进沙箱**：等于递给容器一把宿主机的钥匙，一句话就能逃逸。绝不。
4. **workspace 里的软链接**：symlink 指向 `/etc` 之类路径可能造成穿透。挂载前清理，或镜像里直接禁用。
5. **审批疲劳**：deny list 太粗，天天弹审批，人就会开始无脑点"允许"。策略要细化到误报率可接受，否则防线会被自己人拆掉。

# 可复用建议

- 沙箱一律 `"all"` 起步，验证稳定后再按需放宽；
- 挂载最小化：只挂当前项目目录，用完的挂载及时撤；
- deny list 覆盖 `rm`/`dd`/`mkfs`/`shred` 这类模式，再用 exec 审批兜底；
- 沙箱镜像保持干净：不装 sudo，不留宿主机凭据；
- 定期翻 exec 日志，审计 Agent 实际跑了什么——这不是不信任，是留证据链。

# 总结

OpenClaw 防误删靠的不是 prompt 里一句"请小心"，而是容器隔离、工具策略、审批、职责分离的四层叠加。模型可以继续犯错，但每次犯错的爆炸半径被压进一个可随时销毁重建的容器。对自动化实践者来说，这就是"敢让 Agent 长期跑"的前提：**不是它更聪明了，而是它够不着了。**

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-03/39973189098bbfbf.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-03/855a3c782fd226d2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/a9fb5c2972bebbd9.png)

