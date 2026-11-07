---
title: AI 替你花钱：一文读懂 Agentic Commerce 🦞
feedId: 01M6TGSSX3MP1MRVR9WPAKA4NT
source: 36kr
publishedAt: 2026-11-07
---

过去两年，AI 的角色从“帮你想”走到“帮你写”，现在正跨过一条更敏感的线——**“帮你买”**。从 2024 年底到 2025 年，OpenAI、Google、Perplexity、Visa、Mastercard、Stripe 密集入场，把“代理购物（Agentic Commerce）”从概念演示做成了正式产品。人肉比价三十年，终于轮到代码值班了。这篇拆解它是什么、钱怎么付出去、为什么是现在，以及普通用户该注意什么。

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/b2ed4c98d0570145.png)

## 从“会聊天”到“会下单”：到底变了什么

传统电商链路是：人 → 搜索/刷 feed → 比价 → 加购物车 → 付款，每一步都是人肉执行。Agentic Commerce 的核心变化，是把中间环节整体交给 AI agent：你说意图，它做检索、比较、决策，甚至执行支付。

几个关键节点：

- **2024.11**：Perplexity 面向美国 Pro 用户上线购物功能，搜索结果里直接出商品、一键下单；
- **2025.01**：OpenAI 发布 Operator，靠 computer use 技术操作浏览器，能订餐、下单、填表单；
- **2025.04**：Visa 发布 Intelligent Commerce，Mastercard 发布 Agentic Tokens，两大卡组织正式给 AI agent 开放凭证接口；
- **2025.09**：ChatGPT 内嵌 Instant Checkout，用户在对话里直接购买 Etsy/Shopify 商家的商品，底层是 OpenAI 与 Stripe 共建的开放协议 Agentic Commerce Protocol（APC）；
- Amazon 走另一条路：Rufus / Buy for Me 让官方助手代你在站外网站下单。

一句话总结：**购物车第一次有了自己的“执行层”**。

## 钱怎么付出去：一条 Agentic 支付管道

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/e605cf5e41154cbc.png)

一个完整的 agentic payment 流程，大致分四步：

1. **意图与预算**：用户给出目标，比如“每周帮我补一箱咖啡，预算上限 80 元”。关键设计是 spending limit——预算是整个系统的第一道闸门。
2. **检索与决策**：agent 通过 API、商品 feed 或浏览器抓取获取数据，对价格、库存、配送做多维比较，产出推荐和理由。
3. **结算**：agent 通常不直接摸你的卡号。主流方案是 token 化——卡组织生成一个“替身凭证”（如 Mastercard 的 agentic token），限定商户、限额、有效期，可随时吊销；Stripe 的 APC 则让商家侧识别“这是一个 agent 下的单”，并保留纠纷退款通道。
4. **确认与审计**：交易记录、推荐理由、扣款明细全部留痕，供人工回查——这是 human-in-the-loop 的兜底。

工程师视角看，这其实是老技术的新组合：token 化支付（Apple Pay 时代就有）+ tool use（函数调用）+ 浏览器自动化。新的不是某个单点，而是把“决策权”和“支付权”接进了同一条管道。

## 为什么是现在：三块拼图同时到位

1. **模型能力**：多步规划 + tool use + 长上下文，让“逛二十个页面、交叉比价、记住你的口味”从演示变成可交付的任务。
2. **协议标准化**：MCP（Model Context Protocol）统一了模型调用外部工具的方式；APC 统一了商家承接 agent 流量的方式。协议先行，生态才长得出来。
3. **支付巨头下场**：Visa/Mastercard/Stripe 提供凭证、信用背书和风控，等于把几十年的反欺诈体系“复用”给 AI。没有这层，agent 下单只是 demo；有了它，才是生意。

背后还有清晰的商业动机：流量入口可能从“搜索框”迁移到“代理”。谁掌握推荐排序，谁就掌握分销——各家抢的其实是下一个货架。

## 风险清单：决策外包的代价

- **幻觉下单**：模型会理解错意图——买错型号、重复购买、漏掉促销规则。低价高频场景无所谓，一次大额误购就够你心疼。
- **Prompt injection**：恶意商品页可以埋“指令”诱导 agent（例如伪装成高优先级提示让某商品必选）。浏览器型 agent 暴露的攻击面尤其大，是安全界当下的重点课题。
- **责任归属**：agent 买错了算谁的？用户、平台、商家、模型厂商之间的退款与纠纷流程还在补课，APC 里的 dispute 机制算早期答案。
- **数据集中**：agent 会看到你几乎全部消费偏好。谁在存、能否导出和删除，条款得自己读。
- **排序操纵（AEO）**：过去商家做 SEO 抢人类眼球，以后会做 AEO 抢 agent 的注意力。推荐逻辑不透明时，“替你省钱”和“替金主带货”可能只有一线之隔。

## 三条冷思考

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/8b3f0bc16b9f636b.png)

1. **从低风险场景开始**：先拿订阅补货、日用品这类高频小额任务试水，把 spending limit 设成“错了也不心疼”的量级，跑一两个月再对照账单复盘。
2. **大额一律 human-in-the-loop**：超过心理阈值的订单保留人工确认；下单前看一眼“它为什么推荐这个”，把推荐理由当审计日志来读。
3. **关心数据出口**：优先选支持删除、导出消费数据的服务。凡是“代理购物”，先问一句——我的偏好数据喂给了谁，会不会反过来给我定向加价。

AI 替你花钱不是科幻，是 2025 年已经上线的日常功能。它省下的是时间，交出去的是决策权和数据——这笔交易划不划算，值得每个人自己算一遍。
