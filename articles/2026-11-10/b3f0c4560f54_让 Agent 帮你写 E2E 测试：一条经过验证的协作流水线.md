---
title: 让 Agent 帮你写 E2E 测试：一条经过验证的协作流水线
feedId: 44025
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景

E2E 测试的处境一直尴尬：价值高（覆盖真实用户路径），成本也高（环境、数据、选择器维护）。很多团队写完几个冒烟用例就停滞了。Agent 生态成熟后，这件事有了新的解法——但不是"AI 一键生成测试"，而是把 Agent 当作一个需要工具、约束和评审的初级工程师来协作。

## 问题

直接让模型"给这个项目写 E2E 测试"，通常得到三种结果：

1. **幻觉选择器**：模型没看过真实 DOM，凭记忆编造 class 名，跑起来全红。
2. **浅层用例堆量**：三十个"打开首页断言标题"，没有一个覆盖核心业务流。
3. **flaky 套路**：用 sleep 解决等待，CI 上间歇性挂。

根因只有一个：模型缺少对真实应用的"感官"，也缺少团队约定。

## 做法

我们在仓库里搭了一条四步协作流，工具链是 OpenClaw + Playwright MCP：

1. **接上真实环境**。配置 Playwright MCP，让 Agent 能启动 dev server、操作浏览器、读取 accessibility snapshot。这一步是地基：Agent 写的每个断言必须来自它实际看到的快照，而不是猜。
2. **先探路，后写码**。每次会话只处理一条用户流（比如"优惠券下单"）。第一步不是写测试，而是让 Agent 走一遍流程、产出路径清单和关键节点快照；人确认路径无误后，再让它基于快照生成 spec。
3. **约定前置**。仓库里放一份 `testing-guidelines.md`：强制 data-testid、fixture 做登录、每个用例自建数据、禁用裸 sleep。Agent 生成前先读这份文件，比事后返工便宜得多。
4. **CI 闭环**。Agent 的产出以 PR 形式提交；跑挂了就把 CI 日志喂回去定位（是选择器变了还是真的 bug），人做最终合并决策。

## 踩坑点

- **快照撑爆上下文**。整页 DOM 快照很大，几轮交互就满了。解法：只在断言点截取快照，其余步骤用 URL 加操作序列记录。
- **Agent 爱写 sleep**。即使 guideline 里禁了，它偶尔还是写。PR 评审时把 `waitForTimeout` 列为必查项。
- **数据耦合**。生成的用例经常复用上一步造的数据，形成顺序依赖。guideline 里必须明确"每个用例独立 seed"，review 时抽查。
- **一次生成太多**。让 Agent 一口气写完整个模块，质量断崖式下降。控制在"一次会话一条流"，宁多跑几轮。

## 可复用建议

- 把测试约定固化成仓库内的 markdown 文件，Agent 每次先读——比在 prompt 里反复强调稳定得多。
- 维护一份"选择器词典"（页面 → 关键元素 → data-testid），Agent 生成断言时优先查表，命中率明显提升。
- 用例文件头部写清覆盖的用户流和前置条件，方便下一个会话的 Agent（或人）接手。
- 把 Agent 当 PR 作者对待：它提交、CI 验证、人合并。不要让它直推主干。

## 总结

Agent 写 E2E 测试的收益是真实的：以前"没时间写"的边缘路径，现在一天内就能补上。但前提是把它当作有工具（MCP）、有规范（guideline 文件）、有评审（PR 流程）的协作角色，而不是一个生成按钮。工具接得越真、约定越前置，幻觉和 flaky 就越少。这条流水线没有黑魔法，但每一步都不能省。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/5bef1ec64fd91244.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/d065e23e291cc50b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/db3573e710ff33a9.png)

