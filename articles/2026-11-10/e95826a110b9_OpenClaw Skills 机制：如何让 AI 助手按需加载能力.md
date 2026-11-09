---
title: OpenClaw Skills 机制：如何让 AI 助手按需加载能力
feedId: 43957
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景

OpenClaw 的核心是一个常驻的 Agent 网关，会话启动时会把系统提示词、工具定义和记忆一起喂给模型。能力扩展有两条路：MCP/插件负责接入外部服务，Skills 负责教会模型一套本地流程。Skill 本质上是一份结构化 Markdown 文档加可选脚本，模型自己决定什么时候读、什么时候用。

## 问题

能力少时无所谓，Skill 一多，两个问题同时出现：

1. **上下文膨胀**。如果把每个 Skill 的完整说明都塞进系统提示词，几万 token 起步，成本上升，注意力被稀释，模型反而更容易选错工具。
2. **硬编码不可维护**。把固定流程写死在 prompt 里，每次调整都要改配置、重启、重新验证。

Skills 机制的核心思路是**渐进式披露**：启动时只注入每个 Skill 的 name + description（几十 token），完整正文按需读取，用时才付成本。

## 做法

以一个"周报生成"skill 为例，目录结构：

```
~/.openclaw/skills/weekly-report/
├── SKILL.md
├── scripts/gen.sh
└── references/format.md
```

SKILL.md 分两部分。frontmatter 只保留路由需要的信息：

```yaml
---
name: weekly-report
description: 当用户要求整理本周工作、生成周报或汇总日报时使用。需要 git 和 jq。
metadata:
  openclaw:
    requires:
      bins: ["git", "jq"]
---
```

正文控制在 150 行以内，写三件事：何时使用/何时不使用、快速参考（核心命令与步骤）、常见坑。长文档（如完整格式规范）放 `references/`，脚本放 `scripts/`，让模型需要时自己再读。

生效流程：

1. 目录放到工作区 `skills/` 或全局 `~/.openclaw/skills/` 下，重启网关后用 `openclaw skills list` 确认已索引。
2. 正常对话中提出匹配 description 的请求，翻日志确认模型读入了 SKILL.md。
3. 确定性逻辑尽量下沉到脚本，SKILL.md 里只写"调用哪个脚本、参数是什么、失败怎么办"。

## 踩坑点

- **description 是唯一路由键**。写"帮我处理文档"这种模糊描述永远不会被触发；写成触发条件加排除条件（"仅当用户明确要求 X，不要用于 Y"），命中率明显提升。
- **别把细节全写进正文**。SKILL.md 超过几百行等于换了个地方膨胀上下文，二级文档放 references/ 才是正解。
- **脚本别假设工作目录**。模型可能在任意路径下调用，脚本内用绝对路径或基于自身所在目录推导。
- **requires 门控别省**。没配 bins/env 检查时，依赖缺失的 skill 会被正常加载，然后模型反复调用失败，浪费一整轮会话。
- **改完 frontmatter 要确认生效**。description 的变更不是所有版本都热加载，拿不准就重启网关。

## 可复用建议

- 一个 Skill 只绑定一类任务和一个触发场景，宁可拆小再组合。
- 用 git 管理 skills 目录，版本化和分享都简单；通用能力可以打包发布到 ClawHub 供他人安装。
- 选型经验：本地、确定性、一次性流程用 Skill 脚本；有状态、需鉴权的外部服务走 MCP。两者是互补不是替代。
- 上线后定期翻会话日志，看 skill 是否被触发、触发时读了哪些文件，比凭感觉改 prompt 有效得多。

## 总结

Skills 的本质是把能力写成"模型能路由的文档"：启动时只付路由成本，使用时才付正文成本，这是它与"全量塞进系统提示词"最根本的区别。写好一个 Skill 的标准也很朴素——描述让模型在正确的时机想起它，正文让模型一次读对，脚本让结果可复现。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/9bbf84a7c02529f9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/bdc45d436a0d99e7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/1081f7435784eb76.png)

