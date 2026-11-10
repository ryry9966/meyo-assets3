---
title: OpenClaw 沙箱安全模型拆解：Agent 为什么不会误删你的文件
feedId: 43979
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景

上个月我让 Agent 帮忙清理构建产物，它生成了 `rm -rf ./dist`，但路径解析错了一级，目标实际指向项目上级目录。那次没有损失，纯粹是运气。事后我认真读了 OpenClaw 的沙箱实现，结论很明确：它不靠"模型足够聪明"来保证安全，靠的是分层防线。

## 问题：prompt 约束不是安全边界

LLM 的输出是概率性的。系统提示里写一百遍"不要删除文件"，一次上下文污染、一条诱导性输入、一个插件的错误返回值，都可能让模型生成破坏性命令。所以 OpenClaw 的设计假设是：**Agent 迟早会执行一条错误命令，真正的问题是让这条命令的爆炸半径可控。**

## 做法：四层防线

**1. Workspace 边界。** 文件工具（read/write/edit）默认只能作用于 workspace 目录，这是第一道硬边界，不依赖模型自觉。

**2. Docker 沙箱执行。** 把 sandbox mode 从 `off` 切到 `non-main`，群聊与后台任务的 shell 命令进入一次性容器，主机文件系统不可见，只有 workspace 被挂载。主机路径想进去，必须在 docker bindings 里显式声明——少一条就是不可达。

**3. exec 审批。** 直连宿主的场景下，exec 走审批流：未配对渠道默认 ask，白名单之外的命令要么被拒、要么转人工确认。这层拦的是"看起来合理但你不想要的命令"。

**4. git 快照兜底。** workspace 纳入 git，Agent 改动有 commit 基线，真出事也是可回滚的 diff，不是不可逆损失。

配置大致如下（字段名以你版本 docs 为准，用 `openclaw doctor` 自检）：

```jsonc
{
  "agents": {
    "defaults": {
      "sandbox": { "mode": "non-main", "scope": "agent" }
    }
  },
  "tools": {
    "exec": { "approval": true }
  }
}
```

注意：MCP 和插件提供的工具同样要过 tool policy，不是只有 shell 受控——自制插件的人最容易忽略这一点。

## 踩坑点

- 沙箱开了，但 binding 挂了整个 `$HOME`，等于没挂。挂载范围收窄到任务必需的子目录，能只读就只读。
- 图方便给主会话放行了免审批 exec，又没把 mode 设为 `all`，主会话就成了整条链路最薄的一环。
- 沙箱内默认网络受限。npm/pip 装包失败时先怀疑网络策略，别误判成沙箱坏了直接关掉它。
- 升级后 sandbox 相关字段有变动，`openclaw doctor` 报的 warning 不要跳过。

## 可复用建议

1. 新 agent 一律从 `mode: non-main` 起步，稳定跑两周再考虑放宽。
2. exec 白名单从零条开始，按实际需要逐条加，不要反向裁剪。
3. workspace 必须 git 化，破坏性操作前有基线可回。
4. 定期做对抗测试：故意诱导 Agent 执行删除，观察是哪一层拦住的。没被验证过的防线等于不存在。

## 总结

OpenClaw 的安全模型本质是把"信任模型输出"换成"假设输出会出错"。边界、沙箱、审批、回滚四层叠加后，一条误生成的 `rm -rf` 会被依次缩小范围、隔离执行、人工拦截；即便全部穿透，也只是 git 里一次可回滚的变更。误删从"事故"降级为"日志里的一行"，这才是敢把 Agent 长期交给自动化跑的前提。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/f72288bbdb0c0f78.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/23030b2f3ddeeff3.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/6e6a4c7b7100fad7.png)

