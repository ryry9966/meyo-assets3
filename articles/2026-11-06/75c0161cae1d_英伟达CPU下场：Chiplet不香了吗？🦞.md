---
title: 英伟达CPU下场：Chiplet不香了吗？🦞
feedId: 01M6Q3FV5Z4QB6V68PRQPSN866
source: 36kr
publishedAt: 2026-11-06
---

最近，36kr 等媒体把一个话题顶上了热搜：英伟达在 GTC 上正式端出自研 Arm 架构 CPU「Vera」，与 Rubin GPU 组成 Vera Rubin 平台；同时供应链消息传出，这颗 CPU 可能放弃时下最流行的 Chiplet（芯粒）路线，回归单颗大芯片（Monolithic，单体设计）。于是「英伟达下场做 CPU，还要推翻 Chiplet」的说法开始流传。但在工程师眼里，这更像一道经典的芯片经济学选择题，而不是什么路线斗争。

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/ce172ed6c2de0a67.png)

## Chiplet 到底解决了什么问题

Chiplet 的本质，是把一颗大 SoC 拆成几颗小芯片，再用先进封装拼回去。它换来四样东西：

1. **良率账**：晶圆缺陷密度基本恒定，die 面积越大，命中缺陷的概率越高，而且良率下降是非线性的。把一颗 700mm² 的大 die 拆成两颗 350mm²，废片损失显著减小。
2. **制程混搭**：计算部分用最先进的 N3/N2，I/O 和内存控制器放到成熟制程上，各取所长。
3. **复用与铺货**：AMD Zen 系列的小 CCD 用一套设计铺满桌面、笔记本到服务器全产品线，研发费用摊得极薄。
4. **节奏**：小 die 流片风险低、迭代快，出错也只重做一颗。

代价同样实在：先进封装（CoWoS、硅中介层）不便宜；die-to-die 互连的每 bit 能耗比片内走线高出数倍到数十倍，还多了纳秒级延迟；此外还有 KGD（已知良好裸片）测试成本，以及封装环节新增的良率损失。

## 英伟达的三笔账：Vera 为什么敢做「整芯」

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/908b9e2e2fde00a1.png)

1. **面积与良率账反转**。CPU 的单核面积远小于 GPU。官方口径下 Vera 集成 88 颗自研 Arm 核心；供应链消息称它采用台积电 N3 级工艺、单体设计——die size 控制在单片良率可接受的范围内，拆分的收益覆盖不了新增的封装与互连成本。
2. **产能账**。这两年 CoWoS 是全行业最紧缺的产能，几乎被 HBM 和大 GPU 吃满，而英伟达自己就是最大买家。CPU 若再挤进先进封装产线，等于左手抢右手。单体设计用普通大基板封装，直接绕开瓶颈。
3. **性能账**。D2D 接口带来额外的功耗、面积与延迟，对拼单线程和内存延迟的服务器 CPU 来说，这些开销不像在 GPU 上那样能被并行度掩盖。而且英伟达用的是自研核，吃不到 Arm Neoverse「现成 die」的复用红利——Chiplet 对它本来就少一条理由。

有意思的是，上一代 Grace CPU 恰恰是双 die 通过 NVLink-C2C 拼接的；Vera 改回单体，说明连英伟达自己也在按产品逐颗算账。一句话总结：不是 Chiplet 变差了，是这颗 CPU 的约束条件变了。

## 「推翻」谈不上，边界在移动

- 英伟达本身就是 Chiplet 的最大受益者之一：Blackwell GPU 是两颗逼近光罩极限的大 die 拼合而成，片间 NV-HBI 带宽高达 10 TB/s；HBM 本质上就是 3D 堆叠加 2.5D 互连。单颗 die 做不大的时候，Chiplet 是唯一解。
- AMD 的下一代 EPYC（Zen 6）继续走 chiplet 路线并转向 2nm，Intel 的 Clearwater Forest 则用 Foveros Direct 把 3D 堆叠推上服务器 CPU。
- UCIe 互连标准生态还在持续扩大。业界共识正在变成：2D 拆分解决「die 太大做不了」的问题，3D 堆叠与混合键合解决「带宽和密度」的问题。Chiplet 不是退潮，是往纵深走。

## 商业视角：英伟达要的不是一个 CPU，是整个机架

1. AI 服务器里的 CPU 算力长期被 Xeon / EPYC 分走；自研 Vera 配上 NVLink 互连后，系统利润链收得更拢。
2. 云厂商的自研芯片（AWS Graviton、Google Axion 等）正在分流通用算力需求，英伟达需要用「全栈套装」提高客户绑定度。
3. Vera Rubin 时代的卖点从「一颗芯片」升级为「一个机架」——NVL144 这类 rack-scale 方案里，CPU 哪怕单颗不赚钱，也能帮 GPU 多卖几颗。

所以，「下场做 CPU」这个动作本身，比「用不用 Chiplet」更值得关注。

## 收尾：三条冷思考

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/18e1ef1684872158.png)

1. **技术选型看约束，不看站队**。Monolithic 还是 Chiplet，是一道关于 die size、缺陷密度、封装产能、互连开销的多元方程，答案每年都在变。谁把它讲成路线之争，谁就是在给你讲故事。
2. **看芯片新闻先问三个问题**：这颗 die 多大？先进封装产能排谁前面？D2D 开销吃掉性能几个百分点？三问之后，多数「颠覆」标题会自动降温。
3. **对从业者**：UCIe、先进封装、chip-package-system 协同设计，是十年级别的技能资产；对普通读者，记住一个朴素的工程直觉就够了——好工程师不站队，好工程师算账。
