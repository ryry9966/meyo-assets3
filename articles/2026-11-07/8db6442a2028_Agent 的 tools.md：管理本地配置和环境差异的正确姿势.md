---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 43660
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

OpenClaw 这类 agent 框架里，workspace 下的 `tools.md` 是 agent 每次启动都会读的"本地工具说明书"：这台机器上有什么、装在哪、怎么调、有什么坑。很多人把它当可选的 README 随手写两行，也有人干脆不写，让 agent 每次自己 `which`、`--version` 一路探测。单机上两种做法都勉强能用，但一旦在笔记本、家用服务器、容器三处跑同一个 agent，行为就开始漂移。

## 问题

典型故障基本都见过：

- agent 调 ffmpeg，本机有，服务器没装，报错后它开始自由发挥装一堆东西；
- python 路径写死在 system prompt 里，换台机器 PATH 不同，直接失效；
- 每次会话开头 agent 花好几轮探测环境，token 和时间双输；
- 两台机器上同一个任务行为不一致，排查半天发现是工具版本差异。

根因是同一个：环境"事实"埋在 prompt、记忆或 agent 的猜测里，而不是一份显式、可版本化、可校验的文件。

## 做法

**1. 固定结构，写成指令而非散文。** 每个工具一个区块，字段收敛为四项：路径、版本校验命令、调用模板、已知坑。短句加代码块，agent 读这种格式最省上下文。

```markdown
## ffmpeg
- path: /usr/local/bin/ffmpeg
- check: ffmpeg -version | head -1
- 调用: ffmpeg -i {in} -c:v libx264 {out}
- 注意: 本机无 GPU 编码，禁用 h264_nvenc
```

**2. 环境差异用 delta 表达。** 公共部分放一份共享的 `tools.base.md`，机器差异只写增量：这台装了什么、那台缺什么、路径哪里不同。文件开头明确声明"本机是 XXX 环境"，别让 agent 推断。容器场景用构建脚本从同一份 base 生成，不手工维护两份。

**3. 事实自动生成，判断人工维护。** 写个 dump-env 脚本，扫描关键工具的路径和版本，输出到 `tools.md` 的"自动生成区"；人工只维护注意事项和调用约定，两区之间用注释分隔。这样路径漂移最多导致生成区过期，而不是整份文档烂掉。

**4. 加一道校验。** 启动前或 CI 里跑校验脚本，逐项检查 `tools.md` 声明的路径是否存在、check 命令能否跑通。工具缺失在启动时报错，比让 agent 跑到一半才发现强得多。

**5. 同步走 git，敏感信息进环境变量。** `tools.md` 提交私有仓库或纳入 dotfiles；token、密钥绝不进文件，只引用环境变量名。

## 踩坑点

- 写成了给人看的教程，大段背景介绍，白白消耗上下文；
- 用了 `~` 但 agent 起的是 non-login shell，不展开，路径找不到——验证过的绝对路径最稳；
- bash/zsh 差异：`.zshrc` 里加的 PATH，agent 用 `sh -c` 执行时不可见；
- 版本漂移：Python 3.10 和 3.12 的 CLI 参数可能不同，`tools.md` 必须写版本号；
- 把 `tools.md` 和行为规范文件混用。工具事实和行为约束分开写，各自演化。

## 可复用建议

- 结构模板照抄上面四字段，别自由发挥；
- 文档超过 200 行就拆，agent 不需要知道一切，只需要知道这台机器的事实；
- 探测脚本写一次，本地、服务器、容器构建三处复用；
- 每次装新工具顺手跑一遍 dump 脚本，把维护成本摊薄到日常操作里。

## 总结

`tools.md` 的本质，是把环境事实从 prompt 和 agent 的猜测中外置成一份可版本化、可校验的数据。差异显式化、事实自动生成、启动前校验——做到这三点，同一个 agent 在三台机器上的行为就能收敛到一致。它不解决所有问题，但能消掉你排查环境差异时最常见的那一半时间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/1b46d39502e0f620.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/7b8468cc2ba02753.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/3d7d36f6f7957f91.png)

