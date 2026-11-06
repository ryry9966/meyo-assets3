---
title: OpenClaw Skills 机制：让助手按需加载能力，而不是把所有工具塞进上下文
feedId: 43639
source: 综合讨论
publishedAt: 2026-11-06
---

# OpenClaw Skills 机制：让 AI 助手按需加载能力

## 背景

OpenClaw 的核心形态是一个常驻的 agent gateway：接模型、接渠道、接工具。工具来源很多——内置能力、MCP server、本地脚本。常见的做法是把所有工具的 schema 和说明一次性注入 system prompt。工具少时没问题，一旦挂了七八个 MCP server，仅工具描述就能吃掉上万 token；更麻烦的是，模型在无关场景下也“看得见”这些工具，路由出错率明显上升。

Skills 是这个问题的官方解法：能力以目录为单位组织，每个技能一个 `SKILL.md`，关键设计是**渐进式加载**：

- 启动时只注入技能的 name 和 description，每条几十 token；
- 判断该技能与当前任务相关时，agent 才读取完整正文；
- 正文可以引用同目录下的脚本和文档，脚本建议直接执行，而不是整段读进上下文。

## 问题

之前我在一个网关上挂了多个 MCP server，再把一大段 SOP 写进系统提示，实际体感：

1. 每轮对话固定多消耗一万多 token，成本线性上涨；
2. 模型经常在不该调用的场景调工具，比如让它总结文档，它跑去查日历；
3. 长指令堆在系统提示里，改一次要重启会话，验证也慢。

## 做法

以加一个“周报生成”技能为例：

1. 建目录 `~/.openclaw/skills/weekly-report/`，写入 SKILL.md，frontmatter 至少包含 `name` 和 `description`。description 是路由信号，要写清**什么时候用**：例如“当用户要求整理本周工作、汇总 git 提交或生成周报时使用”。
2. 正文用祈使句写步骤，控制在 100~200 行内；重流程写成同目录脚本（如 `collect.py`），正文只写“执行 `./collect.py`，基于输出生成报告”。
3. 有运行时依赖就在 frontmatter 的 `requires` 里声明（bins、env、os），环境不满足时网关会跳过该技能，避免运行到一半报错。
4. 重启网关或触发重新扫描，确认技能出现在已加载列表；然后实测一句“帮我把这周提交整理成周报”，观察它是否先读 SKILL.md 再动手。

另外，workspace 下的 skills 目录会**覆盖**全局同名技能，适合做个人定制而不污染全局配置。

## 踩坑点

- **description 写得太泛**（比如“帮助处理文档”），结果要么永不触发，要么处处触发。把它当搜索关键词写，放触发场景和关键词。
- **把正文全塞进 description**，等于放弃渐进式加载，token 开销原路返回。
- **脚本输出几百行日志直接打印**，正文里应明确要求截断或汇总后再喂给模型。
- **SKILL.md 里写死密钥**。密钥走环境变量，frontmatter 用 `requires.env` 声明即可。
- **改完技能忘了重新加载**，以为是逻辑问题，其实跑的是旧缓存。

## 可复用建议

- 把 SKILL.md 当 man page 写：短、直接、带示例，不写散文。
- 一个技能只干一件事；复杂流程拆成多个技能组合，别造超级技能。
- 大段参考资料放独立 md 文件，正文里标注“需要时再读”，进一步控制按需加载的粒度。
- 技能放进 git 仓库，symlink 到 skills 目录，团队共享与个人定制两不误。

## 总结

Skills 的价值不在“多了一种插件格式”，而在把**能力声明**和**能力正文**拆开：声明常驻、正文按需、脚本执行。描述即契约，正文即手册。按这个结构组织能力之后，上下文开销和路由准确率的改善是可以直接量化的——对比一下改造前后的每轮 token 消耗就知道了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/8f5aa02ffa7eb83e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/75d34a1e58065ee9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/681c96c1213a6b9c.png)

