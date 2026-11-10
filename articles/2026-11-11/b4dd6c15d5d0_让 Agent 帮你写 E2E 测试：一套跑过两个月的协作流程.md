---
title: 让 Agent 帮你写 E2E 测试：一套跑过两个月的协作流程
feedId: 44058
source: 综合讨论
publishedAt: 2026-11-11
---

## 背景

E2E 测试大概是前端团队里"人人认可价值、没人愿意维护"的部分。手写 Playwright/Cypress 用例成本高，UI 一改 selector 就断，测试目录慢慢变成摆设。我们在 OpenClaw 里接入 Agent 后，把 E2E 用例的初稿生成、失败定位、修复建议三个环节交给 Agent，人工只做审查和验收。跑了两个月，记录一下可复现的做法。

## 问题拆解

直接丢一句"帮我写 E2E 测试"给 Agent，效果通常很差，原因有三个：

1. Agent 对页面真实结构没有感知，只能凭代码猜 selector，生成的用例一半跑不通；
2. 没有断言依据，容易生成"永远通过"的空测试；
3. 一次性产出大量用例，人工审查成本反而超过手写。

所以核心思路是：让 Agent 通过工具拿到页面的真实状态，而不是靠猜。

## 做法

**第一步：接 Playwright MCP。** 给 Agent 挂上 Playwright 的 MCP server，它可以启动浏览器、点击、截图、读取可访问性树（accessibility snapshot）。这是整个流程的地基——Agent 写用例之前先"走一遍"页面，snapshot 里的 role/name 就是天然的语义 selector，比从源码里猜 class 名可靠得多。

**第二步：把测试规范注入上下文。** 将团队的约定写成一份 checklist（selector 优先级、命名、fixture 用法、超时策略），作为 system prompt 或仓库内的约定文件。同时要求 Agent 生成前先读现有用例、模仿风格，避免一条用例一种写法。

**第三步：小批量生成 + 人工审查。** 每次只覆盖一个用户旅程（比如"登录—下单—支付"），产出 3-5 条用例，直接开 PR。审查重点不是代码风格，而是三件事：断言是否真的校验了业务结果、selector 是否走了 data-testid、有没有用 `sleep` 掩盖异步等待。

**第四步：失败用例回流给 Agent。** CI 里测试挂了，把失败日志和 trace 文件喂回去，让 Agent 先给出定位假设——是环境问题、时序问题还是真实 bug——再决定修用例还是提 bug 单。我们要求它必须附带 trace 截图佐证，禁止只改代码不解释。

## 踩坑点

- **别让 Agent 动 CI 配置。** 它为了让测试"变绿"会加 retry、拉长超时，把 flaky 问题藏起来。retry 策略必须人工持有。
- **动态文本导致断言不稳。** accessibility snapshot 里的时间戳、随机 ID 会让断言抖动，需要在约定里明确"哪些字段允许模糊匹配"。
- **上下文长度。** 中型项目一次让它读全部测试文件必炸，按目录分片，每次只注入相关模块。
- **跳过"走页面"这一步，质量断崖式下降。** 初稿质量和第一步强相关，别为了省时间让它直接读代码开写。

## 可复用建议

1. 把 selector 约定做成 lint 规则（禁止 `nth-child`、强制 data-testid），机器约束比 prompt 约定可靠。
2. 给 Agent 一个固定的用例模板让它"填空"，比自由发挥稳定得多。
3. 新用例本地无头模式跑三轮全绿才允许提 PR，把 flaky 拦在合入前。

## 总结

Agent 写 E2E 的价值不在"全自动生成测试"，而在于把最机械的三件事——走页面、起 selector、写等待逻辑——接过去，把工程师的时间留给断言设计和验收标准。工具链就位后，这套流程大约两周能在团队落地。建议先从核心链路切入，别追求覆盖率数字，测试挂了能定位、修完不复发，才是这套协作真正的收益。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/e9044ca0c1b30e00.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/e3c76e6cb6b2c6b6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-11/b50b75d1fbcc6597.png)

