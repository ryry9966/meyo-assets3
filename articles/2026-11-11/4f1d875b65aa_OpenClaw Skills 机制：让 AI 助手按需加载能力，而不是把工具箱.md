---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力，而不是把工具箱全背在身上
feedId: 44050
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景

OpenClaw 的 agent 默认只有一份系统提示词加一组常驻工具。能力一多，问题就来了：MCP server 挂上七八个，每个都往上下文里塞工具描述，token 开销上去了，模型反而更容易选错工具。Skills 是 OpenClaw 给出的一个折中方案：能力以“说明书”的形式存在磁盘上，启动时只登记一行摘要，用到时才完整加载。

## 机制：两级加载

每个 skill 就是一个目录，核心是一个 SKILL.md：

```
~/.openclaw/skills/
└── video-clip/
    ├── SKILL.md          # frontmatter + 使用说明
    ├── scripts/
    │   └── cut.py
    └── references/
        └── bitrate.md
```

frontmatter 里主要是 name 和 description（可选 requires、user-invocable 等元数据）。关键在于加载是两级的：

1. 会话启动时，agent 只拿到所有 skill 的 name + description 组成的紧凑列表；
2. 运行中模型判断任务与描述匹配，才用 Read 工具把完整 SKILL.md 读进上下文，按步骤执行，需要时再通过 Bash 跑 scripts/ 下的脚本。

也就是说，10 个 skill 平时只占 10 行摘要的成本，触发哪个才付哪个的全文 token。

## 做法：写一个能稳定触发的 skill

1. **建目录，写 frontmatter。** description 是唯一的触发面，按“什么情况下用”来写，而不是罗列功能。“当用户要求剪辑、转码或压缩视频文件时使用”，远比“视频处理工具”有效。
2. **正文控制在百行以内**，写清步骤、约束和边界情况；细节资料下沉到 references/，在正文里明确指示 agent 按需再读。
3. **确定性环节脚本化。** 抽帧、转码、格式转换这类活写成脚本放 scripts/，在 SKILL.md 里指示用 Bash 执行，不要指望模型每次心算 ffmpeg 参数。
4. **配 requires 做环境门控**（如 `bins: [ffmpeg]`、`env: [VIDEO_API_KEY]`）。环境不满足时 skill 直接不出现在列表里，避免死技能。
5. **重启 gateway 后做两组验证**：一条应该触发的 prompt，一条相似但不该触发的 prompt，确认触发边界。

## 踩坑点

- **description 含糊**是最常见的问题，直接导致不触发或误触发。它是给模型做匹配用的，不是给人看的简介。
- **SKILL.md 写太长**，每次触发都是一大笔 token。次要内容一律下沉到 references。
- **会话中途新增 skill 不生效**——skill 列表在会话启动时快照，改完要么重开会话要么重启。
- **脚本以你的权限执行。** 第三方 skill 先通读脚本再装，这条没有例外。
- **与 MCP 工具重叠时模型会犹豫。** 经验是把原子操作留给 MCP，把多步工作流沉淀成 skill，二者分工而不是重复。

## 可复用建议

- 团队里重复出现的 SOP（发布检查、日志排查、数据脱敏）都值得沉成 skill，一次写好，所有 agent 复用。
- description 统一用 "Use when ..." 句式，只写触发条件。
- 每个 skill 配一条最小验证 prompt，改动后跑一遍，当回归测试用。
- 定期审计 skills 目录，长期没触发的删掉——描述列表本身也是上下文成本。

## 总结

Skills 不是插件系统，也不是 MCP 的替代品，它更像“延迟加载的说明书”：用一行摘要换一次按需加载，把上下文预算留给当前任务。写好一个 skill 的工作量主要在 description 和脚本化两件事上，其余都是目录约定。先从把一份手头的 SOP 改写成 skill 开始，比一次设计十个更划算。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/00a399b92205cad7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/d0d832b94af7dbc9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/12528f7b5407b502.png)

