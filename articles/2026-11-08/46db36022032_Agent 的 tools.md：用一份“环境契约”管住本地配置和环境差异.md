---
title: Agent 的 tools.md：用一份“环境契约”管住本地配置和环境差异
feedId: 43809
source: 综合讨论
publishedAt: 2026-11-08
---

## 背景

跑 Agent 做自动化（OpenClaw、MCP server、各种插件）时，最磨人的往往不是模型能力，而是环境：每台机器的工具链、命令名、路径、包管理器都不一样。同一个任务，A 机器 `python` 能跑，B 机器只有 `python3`；这台有 Docker，那台没有；公司内网还要走代理。这些差异最后都变成了对话里一句句的口头纠正，换台机器重来一遍。

## 问题

常见三种做法都有毛病：

1. **全写进系统提示词**：臃肿，且换机器、换项目就要改提示词；
2. **靠 agent 现场探测**：每次浪费好几轮工具调用，还可能得出错误结论后一路错下去；
3. **口头同步**：换人、换机、长会话失效，等于没有。

缺的是一份机器可读、随仓库走、又允许机器级覆盖的“环境事实清单”。这就是 `tools.md` 要解决的。

## 做法：两层结构

约定两个文件：

- `tools.md`：仓库级，提交进 git，写**团队公共基线**——任务依赖哪些工具、目录约定、禁区；
- `tools.local.md`：机器级，加入 `.gitignore`，写**这台机器的事实**——实际命令名、路径、版本、代理。

在 agent 的启动指令里只放一句话：开工前先读这两个文件，冲突时以 local 为准。

示例片段：

```markdown
# tools.md（仓库级，提交）
- 任务入口：./scripts/run.sh
- 测试：先跑 pytest -x，失败即停，不要跳过
- 禁止：全局 pip install、写 /etc、执行 sudo

# tools.local.md（机器级，gitignore）
- Python 命令：python3（本机无 python 别名）
- 包管理：brew；apt 不可用
- 工作区：~/work/oc-demo
- 代理：仅 git/curl 需要，读环境变量 HTTPS_PROXY
```

核心原则：**描述“是什么”，而不是“应该是什么”**。写成可执行的事实，不写设计散文。

## 防漂移：定期校验

环境会烂，文件会过期。写一个十几行的探测脚本：检查命令是否存在、路径是否可达、版本是否符合，输出与 `tools.local.md` 的 diff。可以挂 pre-commit，也可以在会话开头让 agent 跑一次。发现不一致，改文件，而不是改提示词。

## 踩坑点

1. **秘密进仓**：API key、内网地址写进 `tools.md` 还 commit 了。文件里只写变量名引用环境变量，值永远不落盘。
2. **写死绝对路径**：`/Users/xxx/...` 换机即失效，用 `~` 和仓库相对路径。
3. **与 MCP 重复维护**：MCP server 的 schema 是工具接口，`tools.md` 是环境事实。参数说明别两边都写，`tools.md` 只记录“哪个 server 在这台机器可用、启停命令”。
4. **长会话缓存**：改了文件，agent 上下文里还是旧版，误以为“改了没用”。改完重开会话，或显式要求重读。
5. **分层失衡**：local 文件忘记 gitignore，把个人路径推给全组；或者反过来只有 local，新同事没有基线，第一天就踩雷。

## 可复用建议

- **固定命名**：`tools.md` / `tools.local.md`，让约定不需要解释；
- **固定章节**：机器画像、工具清单、路径、禁区、差异与回退，五段足够；
- **探测脚本沉淀成 `tools doctor`**：新人入职先跑一遍，输出即体检报告；
- **换机、升级大版本后主动 review**，别等 agent 报错才发现文件过期了。

## 总结

`tools.md` 的价值不在于“又多一个文档”，而在于把环境差异从“对话里的口头知识”变成“随仓库版本化的契约”：仓库级定基线，机器级管差异，探测脚本防漂移。做完这三件事，同一套 agent 流程换台机器，基本可以做到少纠正甚至零纠正——这才是自动化该有的样子。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/208b335be1a2b2fc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/077fff5d02c68cca.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/a4fce0717794ffea.png)

