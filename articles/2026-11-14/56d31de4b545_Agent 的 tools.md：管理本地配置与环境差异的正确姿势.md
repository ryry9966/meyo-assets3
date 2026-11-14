---
title: Agent 的 tools.md：管理本地配置与环境差异的正确姿势
feedId: 44420
source: 综合讨论
publishedAt: 2026-11-14
---

## 背景

跑过 Agent 的人大概率遇到过这个场景：同一套 skill 和 MCP 工具，在 A 机器上一次跑通，换到 B 机器就报“找不到命令”“环境变量未设置”。原因不复杂——工具的接线信息散落在 shell rc、.env、MCP 配置 JSON、插件注册代码里。Prompt 和 skill 能 git 共享，但“这台机器上工具怎么装、怎么起”往往只存在于每个人的脑子里。

## 问题

散落式配置有四个典型症状：

1. **绝对路径硬编码**。工具描述里写着 `/home/xxx/venvs/rag/bin/python`，换人换机即失效，且失败发生在 Agent 调用那一刻，排查成本高。
2. **多处事实不一致**。MCP 配置说工具在端口 A，.env 写端口 B，插件代码里还有默认值 C，Agent 拿到哪个全看加载顺序。
3. **秘钥与结构混在一起**。想共享配置结构，但文件里夹着 key，只能整份 gitignore，接手全靠口口相传。
4. **配置漂移无人发现**。升级了 Python、挪了工作目录，没有机制提前报警，直到任务中途炸掉。

## 做法：一份 tools.md + 一份 local 覆盖

核心思路：把“工具怎么接线”收敛成一份机器无关的 `tools.md` 作为唯一事实源，机器相关的值抽到不入库的 `tools.local.md`，启动前用 preflight 脚本校验。

**Step 1 盘点**：grep 一遍现有的 MCP 配置、.env 和启动脚本，列出每个工具的启动方式、依赖和路径。

**Step 2 写 tools.md**，结构固定、字段克制：

```markdown
## tool: rag-retriever
- launch: ${PY_BIN} ${TOOLS_DIR}/rag/server.py --port ${RAG_PORT}
- env: PY_BIN, TOOLS_DIR, RAG_PORT, RAG_API_KEY(ref)
- requires: python>=3.11, faiss
- health: curl -sf localhost:${RAG_PORT}/health
- verified: 2025-06-30
```

三条硬约束：env 只写变量名不写值；路径全部走变量；每个工具必须带一条 health 命令。

**Step 3 tools.local.md 只放值**。`PY_BIN=/usr/local/bin/python3.11` 这类机器赋值，gitignore 掉。加载顺序固定为 tools.md 在前、local 覆盖在后，preflight 里显式打印实际生效值，避免静默失效。

**Step 4 preflight 脚本**：逐项检查二进制存在、版本满足、变量名齐全、health 可通，输出一张表，红了就停在启动前。挂在 Agent session 启动之前，或放进 CI。

**Step 5 让 Agent 读 tools.md**：工具发现指向解析后的这份文件，描述、启动命令、健康检查收在一处，调用失败时也有据可查。

## 踩坑点

- 手滑把 key 的值写进 tools.md 并提交了。补救：preflight 加一条规则，env 字段出现字面值即报错。
- local 覆盖没生效，多半是加载顺序写反，或变量名大小写不一致。打印“生效值来源”能一眼定位。
- tools.md 写成散文。它是给 Agent 和脚本消费的，字段要短、结构要稳，长解释放 wiki，文件里留一行链接。
- health 用了 macOS 独有命令，Linux 上 preflight 全红。尽量用 curl、python 这类跨平台原语。
- 漂移重新累积。`verified` 超过一个月未更新的条目，preflight 给黄灯提醒重验一次。

## 可复用建议

- 仓库 `templates/` 下放一份 `tools.md.example`，新人复制改名即可上手。
- 只 review tools.md，local 永不入库，两者字段结构保持同构。
- 安装脚本与 tools.md 分离：前者管“装”，后者只描述“接”。
- preflight 输出留档，环境问题从“任务中途报错”变成“启动前一屏看全”。

## 总结

tools.md 本身只是百来行 markdown，价值在于把环境差异从运行时的意外，压缩成启动前的一次校验。工具接线有了唯一事实源，Agent 的失败才有稳定的排查起点。这套做法没有黑科技，难的从来不是文件格式，而是团队愿意把散落的配置收进一份清单，并坚持维护下去。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/ef69bf5d694778f8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/c90717d6b014ef22.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/59a0cab6b0ff208f.png)

