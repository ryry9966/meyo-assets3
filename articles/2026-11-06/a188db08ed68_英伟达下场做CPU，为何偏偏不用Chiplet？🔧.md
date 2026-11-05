---
title: 英伟达下场做CPU，为何偏偏不用Chiplet？🔧
feedId: 01M6Q2367G439HH04HWVPB4HNG
source: 36kr
publishedAt: 2026-11-06
---

最近一轮热议来自英伟达的 GTC 发布会：下一代 Vera Rubin 平台里，除了 Rubin GPU，还首次出现了英伟达自研的 Vera CPU——88 个 Arm 架构核心。真正的讨论点不在核数，而在封装形态：这颗 CPU 是一块完整的单 die（monolithic），没有跟进行业的 chiplet 主流。AMD、Intel、Arm 推了多年的路线，要被英伟达掀翻了吗？先把问题拆开看。

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/422a483a8549579c.png)

## 先补课：Chiplet 到底在解决什么问题

Chiplet（小芯粒）的思路是：把一颗大芯片拆成多颗小 die，分别流片，再用先进封装拼回来。概念脱胎于更早的多芯片模块（MCM）思路，经 DARPA 的 CHIPS 项目推动后有了今天的形态，最终在 AMD 手里被大规模验证。它要解决的是四个硬问题：

1. **物理上限**：光刻机单次曝光的掩模版视场有限（约 858mm²），单 die 面积迟早撞墙；
2. **良率经济**：die 越大，落进缺陷的概率越高，一颗 800mm² 的 die 良率可能远低于两颗 400mm²；
3. **工艺混搭**：计算核用最先进节点，I/O 与模拟电路用成熟节点，成本结构完全不同；
4. **IP 复用**：一颗设计好的小 die 可以跨产品线反复使用，摊薄先进制程动辄上亿美元的研发与流片投入。

先例也够多：AMD 自 Zen 2（2019）起全面 chiplet 化，用 Infinity Fabric 打通跨 die 缓存一致性；Intel 从 Meteor Lake 起转向 tiles 设计；2022 年 Intel、AMD、Arm、台积电、英伟达等成立 UCIe 联盟，统一 die 间互联标准。但 CPU 一直是最难 chiplet 化的那一个：缓存一致性协议复杂，跨 die 一次内存访问多出数十纳秒的额外时延，会被每个核心放大成整体性能损失。

## Vera 的反常选择：88 核，单 die

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/6dce0406949010a8.png)

- **形态**：88 个英伟达自研 Arm 核心（不再沿用 Grace 上的 Neoverse 公版核），官方明确为 monolithic 单 die 设计；
- **理由一：延迟**。跨 die 互联的每一跳都增加功耗与时延，而 Vera 要通过 NVLink 与 Rubin GPU 做缓存一致性耦合，CPU 内部再拆 die 等于叠加两层边界；
- **理由二：规模合适**。88 核在先进制程下没有触及单 die 的物理极限，拆分的收益覆盖不了代价；
- **理由三：封装账**。单 die 省去 2.5D/3D 先进封装的额外工序、良率损耗和供应链依赖；
- **对照**：同平台的 Rubin GPU 本身就是 chiplet——两颗 reticle 级 die 加 HBM4。同一家公司、同一个平台，GPU 拆、CPU 不拆，说明这是按产品约束算出来的选择，不是路线表态。

顺带一提，核数从 Grace 的 144 降到 88，换来自研微架构的灵活度：单核更强、与 GPU 协同更深。

## 英伟达的逻辑：它卖的不是 CPU，是机柜

- Vera CPU 目前不单独零售，而是作为 Vera Rubin NVL144 机柜的一部分，与 GPU、NVLink 6、网络一起打包交付；
- 英伟达的商业模式是全栈垂直整合：GPU + CPU + 互联 + 网络 + CUDA 软件栈，客户按机柜规模与 token 吞吐付费；
- 在这个体系里 CPU 是配角：不需要像 AMD、Intel 那样靠堆核数去打单芯片性能榜，只需要把延迟、功耗和 GPU 协同做到极致；
- 更大的背景是，云厂商的自研 Arm CPU（AWS Graviton、Google Axion、Microsoft Cobalt）已经把数据中心 CPU 的话语权从 x86 双雄手里分走了一部分。英伟达此时入场，卖的是"整套系统里最顺滑的那颗 CPU"，而非通用处理器。

从这里看，所谓"反潮流"并不奇怪：chiplet 的三大红利——良率、混工艺、IP 复用——主要服务于"需要单卖 CPU"的厂商。英伟达的 CPU 不单卖，这些红利吸引力有限，而 chiplet 的成本（die 间互联、封装、生态磨合）却是实打实的。

## Chiplet 被推翻了吗？没有，只是边界变清晰

1. **Chiplet 是工程手段，不是信仰**。它有明确适用条件：die 大到良率损失不可接受、需要混用工艺、需要复用 IP。三条都不占，单 die 更优；
2. **核心数是分水岭**：88 核单 die 划算；如果目标是 200 核以上加大缓存，跨 die 拆分的收益曲线就会反转。Vera 的选择不代表所有服务器 CPU 都该照抄；
3. **行业主线未变**：Rubin GPU 自己就是 chiplet，AMD、Intel 的服务器 CPU 会继续走 chiplet，UCIe 生态照常扩张——英伟达自己还是 UCIe 的创始成员；
4. **真正的贡献**，是把 chiplet 的隐性成本摆上台面：die 间互联的功耗与延迟、封装测试开销、多供应商磨合难度。英伟达等于做了一次公开算账的示范。

一句话：Vera 不是 chiplet 的墓志铭，而是提醒所有人——chiplet 是工具，不是路线图上的必经站。

## 给从业者的三条冷思考

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/5e2beaea57a94677.png)

1. **选型看约束，不看风潮**。动手之前先算三笔账：预计 die 面积与良率曲线、互联延迟对端到端性能的影响、先进封装的成本与产能，再决定拆不拆；
2. **把 chiplet 的隐性成本写进评估表**：die-to-die PHY 功耗、封装良率、测试覆盖、多供应商磨合成本。很多人只算了 wafer 那一栏；
3. **盯住价值转移的信号**：竞争单位正从"单颗芯片"变成"机柜级系统"，性能口径从单核跑分变成整柜 token 吞吐和每瓦性能。做 CPU、做加速器、做软件的团队，都值得重新核对自己的定位。

在芯片行业，风潮是给别人参考的，账是给自己算的。Vera 这颗"不拆分"的 CPU，最大的价值就是帮所有人把账本翻到了 chiplet 那一页。
