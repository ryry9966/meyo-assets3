---
title: OpenClaw Skills 机制：让 AI 助手按需加载能力
feedId: 44369
source: 综合讨论
publishedAt: 2026-11-14
---

## 背景

接了七八个 MCP server 之后，我最大的感受不是能力强了，而是系统提示词变"重"了：每个会话都要背着全部工具 schema 和说明跑，模型选工具时也容易漂。OpenClaw 的 Skills 机制走的是另一条路——能力不常驻，而是像书架上的说明书，按需取用。

核心是渐进式披露（progressive disclosure）：gateway 平时只把每个 skill 的 `name` + `description` 注入上下文，成本几十个 token；模型判断当前任务相关时，才读取完整的 SKILL.md 正文，再按需加载 skill 附带的脚本和参考文件。三层结构，逐层付费。

## 问题

我自己的场景：把部署 SOP、日志巡检、告警处理这些流程写进常驻 system prompt，约 3000 token，每个会话全额支付，但大部分会话根本用不到。与此同时 MCP 工具越挂越多，误选率上升。目标很明确：把"流程性知识"从常驻上下文里挪出去。

## 做法

1. 在 `~/.openclaw/skills/` 下建目录，每个 skill 一个文件夹，入口是 SKILL.md：

```markdown
---
name: deploy-check
description: 当用户要求发布、回滚或检查线上版本状态时使用。覆盖 web 与 worker 两个服务。
---

# 发布检查
1. 运行 scripts/precheck.sh 确认 CI 状态
2. 回滚步骤见 references/rollback.md
3. 不得直接操作生产数据库
```

2. `description` 决定触发，写"何时用"，不要写"这是什么"。
3. 正文写步骤和边界条件；超过一屏的细节拆到同目录 `references/*.md`，正文只留文件索引。
4. 脚本放 `scripts/`，用 frontmatter 声明 `requires.bins` / `requires.env`，环境不满足时该 skill 会被跳过。
5. 验证：`openclaw skills list` 确认加载，`openclaw skills info <name>` 检查描述与依赖；然后对 agent 说一句触发话术，观察它是否去读 SKILL.md。
6. 按需开关：在 `openclaw.json` 里用 `skills.allow` / `skills.deny` 控制，也可对单个 agent 覆盖；团队共享的 skill 建议走 git 仓库统一分发。

## 踩坑点

- **description 写成名词解释**："这是一个部署工具"基本不触发。写成条件句："当用户要求发布/回滚上线时使用"，命中率立刻不同。
- **正文贪长**：把整本 runbook 塞进 SKILL.md，触发一次吃 2000 token，等于换个地方付钱。正文控制在 150 行内，细节进 references。
- **撞名**：与内置 skill 同名会导致加载和覆盖关系混乱，自定义 skill 加前缀（如 `team-deploy-`）。
- **路径假设**：脚本里写相对路径，而 agent 会话的 cwd 不确定，改用基于 skill 根目录的显式路径。
- **第三方 skill 是注入面**：从 ClawHub 装别人的 skill 前通读全文，警惕脚本里的 `curl | sh` 和硬编码的环境变量。
- **改完不生效**：skill 列表在 agent 侧有快照，多 agent 场景下改完 SKILL.md 记得确认快照刷新或重启 gateway。

## 可复用建议

- **分工原则**：MCP 管"协议连接"（数据库、消息、长连接 API），skills 管"流程知识"（怎么发版、怎么值班、怎么写复盘）。不要用 skill 去复刻一个已有的 MCP 工具。
- **description 模板**：触发条件 + 动作 + 适用范围，一到两句。
- **一个 skill 一件事**：超过 150 行就拆，粒度细反而更容易被正确触发。
- **SOP 进 git**：流程知识以 skill 形式版本化、走 review，比散落在群公告里的文档可靠得多。
- **量化收益**：对比开启前后的上下文占用（`/status`），通常能看到系统提示词明显缩水，这就是省下来的钱。

## 总结

Skills 的本质不是插件，而是按需加载的说明书。它把 token 成本从"每个会话全额预付"变成"用到的才付"，把流程知识从 prompt 工程变成文件工程。MCP 负责连接，skills 负责流程，两者各归其位——这是目前我用下来最省心的 agent 扩展组合。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/ebe8f733541ddde2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/ff0444bccb99ab4a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/3710086c726294c4.png)

