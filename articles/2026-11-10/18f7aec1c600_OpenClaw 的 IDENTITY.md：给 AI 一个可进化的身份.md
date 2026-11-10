---
title: OpenClaw 的 IDENTITY.md：给 AI 一个可进化的身份
feedId: 43985
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景

OpenClaw 的 agent 行为不靠硬编码，而是靠 workspace 里一组 Markdown 文件引导：`AGENTS.md` 管操作规程，`SOUL.md` 管性格与边界，`IDENTITY.md` 管"我是谁"。身份卡记录名字、形象、语气约定、emoji 用法，随系统提示常驻注入每个会话。只要它存在，跨会话才有稳定的"同一个人"。

## 问题

没有 identity 或乱写 identity，常见症状是：

- 每换个会话，自我介绍就变一次，称呼忽冷忽热；
- 同一个 gateway 挂多个 agent，人格互相串；
- 想调语气只能改全局提示词，改一次全量生效，且无变更记录；
- 性格指令散落在配置各处，时间一长没人说得清"它现在是什么人设"。

本质问题：身份是一个持续变化的 artifact，却没被当作 artifact 管理。

## 做法

1. 在 workspace 根目录（默认 `~/.openclaw/workspace/`）建 `IDENTITY.md`，控制在 10 行以内：

```markdown
# IDENTITY
- Name: 小爪
- Creature: 一只安静的机械猫头鹰
- Vibe: 务实、简短，先结论后步骤
- Emoji: 仅在完成多步任务后用一个
- 边界: 不替用户执行不可逆操作
```

2. 与 `SOUL.md` 划清边界：IDENTITY 管"名片"，SOUL 管"价值观与红线"。重复内容删掉，冲突时以 SOUL 为准。
3. 打通进化回路：把 workspace 纳入 git。约定一条规则——对话里你重复纠正超过两次的点（语气、称呼、格式），沉淀成一行写回 IDENTITY.md，commit 留痕。
4. 反向通道：agent 在 heartbeat 或记忆整理时可以"提案"身份修改，落到单独的提案文件，你 diff 后手动合入，禁止直接自改。
5. 验证：改完后问三个固定问题（自我介绍、复杂任务汇报、拒绝场景），对比输出风格是否如预期变化。

## 踩坑点

- **写长了**：IDENTITY 常驻 system prompt，几千字的身份卡既费 token 又稀释关键指令。我的版本最终稳定在 8 行。
- **塞事实记忆**：项目上下文、联系方式这类内容应进 `USER.md` / `MEMORY.md`，混进 identity 只会让它无限膨胀。
- **免审自改**：放开过一次让 agent 直接编辑，一周就漂移到过度拟人，靠 git 回滚才救回来。
- **与 AGENTS.md 冲突**：emoji 规则在两处写得不一致，模型输出随机偏向其一，收敛成单一来源才稳定。

## 可复用建议

- 把 identity 当代码管理：小、可 diff、可回滚；
- 变更来源固定：只收录真实对话里的纠正点，不凭空想象；
- 多 agent = 多 workspace，各自 identity，互不污染；
- 每次只改一行，改完跑固定测试问题，保持小步迭代。

## 总结

IDENTITY.md 的价值不在文件本身，而在于把"人格"变成可版本化、可评审、可回滚的工程对象。一行身份描述的影响远比想象中大。管好这个几 KB 的小文件，agent 才能长期一致地做"同一个助手"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/55ed714ced0d9a9b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/520899c24318f102.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/a5f808709f3a8da0.png)

