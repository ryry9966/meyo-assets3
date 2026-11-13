---
title: 从 Timsort 到 Python 之禅：把审美写进一门语言的人 🐍
feedId: 01M7A9C0914DDEE5QQ3DNQEH6N
source: 36kr
publishedAt: 2026-11-13
---

据 36kr 等媒体报道，《Python 之禅》作者、Python 核心开发者 Tim Peters 逝世。Guido van Rossum 发文悼念，其中一句话在技术圈刷屏：没有他，就没有今天的 Python。不少开发者的第一反应是——这名字有点陌生。但只要你写过 Python、Java 或 JavaScript，你的程序大概率每天都在跑他写的算法、遵守他定的“规矩”；说你的 CPU 比你本人更熟悉他，并不夸张。这篇文章把他留下的技术遗产拆开讲清楚。

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/ef0600eccf3693b9.png)

## 一、他是谁：不是创始人，却是“半个缔造者”

- Tim Peters 是美国程序员，上世纪 90 年代起活跃在 comp.lang.python 新闻组，以回复长、细节密著称：别人回帖三行，他动辄写一篇小论文，而且几乎句句有信息量。
- 他的身份很特殊——不是语言“掌门人”，大部分贡献以社区开发者身份完成，早年曾在 Zope 公司任职。可以说，Guido 决定 Python “是什么”，而 Tim 更像是把 CPython 从个人项目打磨成工业级产品的关键工匠。
- Python 2.x 时代，他贡献了大量核心代码、性能优化和设计讨论。Guido 悼念里那句“没有他就没有今天的 Python”，指的主要就是这段长期、沉默的工程投入。

## 二、Timsort：被写进行业标准的排序算法

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/5e0ef941db0a6338.png)

- 2002 年，Tim 为 CPython 重写了 `list.sort()`，这套实现后来被命名为 Timsort——直接冠他自己的名字，开源圈少见的“实至名归”。
- 核心思路一句话：真实世界的数据不是教科书里的随机数组，往往自带大量已有序的片段。Timsort 会扫描数据找出这些有序“run”，小段用插入排序补齐，再用归并排序逐层合并；如果数据本来就基本有序，耗时接近线性 O(n)。
- 三个特性同时成立：稳定排序（相等元素保持原顺序）、自适应、最坏情况 O(n log n)。这三条凑齐，在排序算法里相当难得。
- 外溢效应惊人：Java 7 起，对象数组排序默认改用 Timsort；Android、JavaScript 引擎 V8、GNU Octave 等先后跟进。一个为 Python 写的算法，成了整个软件行业的事实标准。
- 顺带解释一个日常现象：Python 的 `sorted()` 保证稳定，所以“先按 A 排、再按 B 排”的多级排序套路才成立。这份保证的底层，正是 Timsort。

## 三、Python 之禅：19 句话如何塑造一门语言的品味

- 1999 年，Tim 在社区写下 The Zen of Python，2004 年正式收录为 PEP 20。今天在 REPL 里敲 `import this`，就能看到全文。
- 全文只有 19 条格言：Beautiful is better than ugly、Simple is better than complex、Readability counts、Errors should never pass silently……每条都是短句，组合起来却是一整套设计价值观。
- 定位要分清：它不是编码规范（那是 PEP 8 的职责），而是 PEP 8 背后的“立法精神”。PEP 8 里几乎每条规则，都能在 Python 之禅里找到出处。
- 著名彩蛋：`this` 模块的源码本身是 ROT13 加密的，打开一看全是乱码——用违反自己原则的方式陈述原则，堪称程序员世界里最优雅的一次“自我打脸”。
- 还有条冷知识：之禅只有 19 条，第 20 条一直空着，留给 Guido，也留给社区，至今没人补上。留白本身成了遗产的一部分。

## 四、看不见的贡献：doctest、timeit 与一次关键提速

- `doctest` 模块出自他手：把交互式示例直接写进 docstring 当测试用例，“文档即测试”。这个思路影响了后来的整个文档测试文化。
- `timeit` 模块也是他的作品：今天你测“这两行代码哪个快”，用的还是它，它甚至默认关闭 GC 来避免测量干扰——细节控到这个程度。
- 更硬核的是性能优化：Python 2.3 前后，他把函数本地变量的存储从字典查找改成数组索引（fast locals），函数调用开销大幅下降；小整数缓存、对象复用等内存策略也大量出自他手。CPython 在那个版本前后“快了一大截”，功劳簿上他的名字密度极高。
- 另外还有 `isinstance()` 支持传入 tuple 这类 API 改进，以及海量的 code review。Guido 的悼词不是客套，翻开 CPython 变更日志就能验证。
- 番外身份：他在 Stack Overflow 算法话题下的长答案，至今仍是很多人学习排序与复杂度分析的参考材料。

## 五、冷思考：一个人能给一门语言留下什么

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/fc2b2b06f768f249.png)

1. 品味可以被工程化。“代码要优雅”是句空话，但 "Readability counts" 是可以讨论、执行、review 的原则。把主观审美翻译成可操作的短句清单，是他留给所有技术团队的方法论。
2. 开源生态依赖少数“沉默的长期主义者”。我们天天调用的基础设施，背后往往是不求曝光的贡献者。可持续的开源不只是募捐，还包括认真 review、写文档这类不性感但关键的事。
3. 为真实数据设计，而不是为教科书设计。Timsort 赢过无数理论上更漂亮的方案，靠的是对真实数据形态的观察。工程判断里，“贴合现实”经常比“理论最优”更值钱。

## 三条实用建议

- 今天就在终端敲一次 `import this`，然后打开 `this.py` 源码读一遍——几十行，看完你会对“用反例讲原则”有新的体会。
- 把 PEP 20 当 code review 的锚点：看到裸的 `except: pass`，引用 "Errors should never pass silently"；看到一行塞三个逻辑，引用 "Readability counts"。有出处的意见，说服力完全不同。
- 想学“算法如何工业化”，去读 CPython 源码 `listobject.c` 里 Tim 留下的长注释，观察一个论文算法如何落地：边界处理、内存控制、退化情形兜底，全是教科书不写的实战细节。

Guido 造出了 Python 的骨架，Tim Peters 给了它速度和品味。下次 `import this`，屏幕上会自动打印那 19 行——那是他留在这个行业里、不会过期的签名。
