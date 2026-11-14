---
title: 给 Agent 一份 tools.md：把本地环境差异变成可读的事实
feedId: 44376
source: 综合讨论
publishedAt: 2026-11-14
---

## 背景

跑本地 Agent（OpenClaw 或任何带 MCP/插件机制的工具型 Agent）最常见的浪费，不是模型不够聪明，而是它每次会话都在重新认识你的机器：在 macOS 上想用 apt、分不清 `python` 还是 `python3`、Docker 没起、代理没配。这些问题反复重演，消耗 token，偶尔还会造成破坏性操作。

## 问题

环境信息散落在 `.zshrc`、MCP 配置 JSON、各种 dotfile 里，Agent 看不见。它只能靠猜测加试错，而试错在本地环境是有代价的。团队场景更糟：同一份 prompt 在三台机器上行为不一致，排障全靠口口相传。

## 做法

核心思路：把「这台机器的事实」固化成工作区根目录的一个 `tools.md`，让 Agent 引导阶段先读它。

建议结构：

```markdown
# tools.md
## 系统
- macOS 14 / Apple Silicon / 包管理用 Homebrew（不要用 apt）
## 运行时
- python3.12（pyenv 管理，默认 venv 在 .venv）
- node 20（nvm）
## 常用路径
- docker 走 colima，需先 `colima start`
## 网络
- 代理 http://127.0.0.1:7890，pip/npm 已配置
## 禁止
- 不要全局 pip install
- 不要改 /etc/hosts
```

落地步骤：

1. **初稿用脚本生成，别手写。** 导出真实的版本、路径、包管理器，避免写进一厢情愿的内容。
2. **按上面结构裁剪**，只保留 Agent 会用到的条目。
3. **接入引导**：在 AGENTS.md 写明「涉及本地命令的操作，先读 tools.md」。
4. **分层管理**：机器相关的放 `tools.local.md` 并 gitignore；团队通用部分进 git。
5. **建立反馈回路**：Agent 因环境问题失败一次，就往 tools.md 补一行。这是文件长期保持准确的关键。

## 踩坑点

- **别写期望状态。** 文件里写了 Docker 但实际没装，Agent 会盲信然后翻车。初稿必须来自真实导出，并定期用脚本核对。
- **别放密钥。** tools.md 会整份进入上下文，密钥只写变量名，值留给环境变量。
- **别写太长。** 这是上下文预算，控制在几百行内，失效条目定期清理。
- **和 AGENTS.md 划清边界**：AGENTS.md 管行为约定（怎么做事），tools.md 管环境事实（这台机器有什么）。混写会失控。
- **WSL/Windows 用户**要写清用哪个 shell、路径按哪套体系解析，否则 Agent 会在两种路径风格之间反复横跳。

## 可复用建议

- 写一个 `tools-doctor.sh`：导出当前版本信息，同时校验 tools.md 里登记的路径是否真实存在，输出 diff。
- 允许 Agent 提议修改 tools.md，但必须走 diff 审查，不许静默自改。
- 验收方式很简单：开新会话直接问「这里该用什么包管理器、python 怎么调」，答对且零探测命令，说明文件已生效。

## 总结

tools.md 成本极低，收益明确：更少的试错 token、更少的破坏性操作、团队环境可复现。本质是把本地环境当作接口来文档化——Agent 的可靠性上限，往往取决于你喂给它的「事实」的质量。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/b987495ed5fb7bfb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/718c1976a4975102.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-14/b8c5825d570f18eb.png)

