---
title: LLM 输出解析的防御性编程：混合 JSON 格式的分层提取实践
feedId: 44267
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

在 OpenClaw 的插件和子 Agent 流程里，让模型返回结构化 JSON 是家常便饭：MCP 工具消费上游输出、自动化流水线节点间传数据、插件解析模型汇报。实践下来最稳定的规律反而是——模型不会按你期望的格式稳定输出。同一段 system prompt，昨天返回裸 JSON，今天包一层 ```json 围栏，明天变成 `<result>{...}</result>`，偶尔前后还各夹一句解释。

## 问题

裸调 `JSON.parse` 的失败率随模型、温度、prompt 长度波动。很多人的第一反应是加正则，然后正则又被嵌套大括号、多代码块、全角引号反复打穿。解析逻辑散落在各插件里，等于每个插件把同样的坑踩一遍。

## 做法：分层解析流水线

核心思路：不猜格式，按可信度优先级逐层尝试，命中即返回，全部失败再走修复与重问。

1. **直接 parse**：快路径，覆盖理想情况；
2. **围栏提取**：匹配代码围栏块，语言标记（json/JSON/json5）忽略大小写；
3. **标签提取**：对 json / result / answer / output 等候选标签名做大小写不敏感匹配；
4. **括号配对扫描**：从第一个 `{` 或 `[` 起做状态机式配对（正确处理字符串与转义），截出首个平衡块——这里不要用正则；
5. **容错修复**：去尾逗号、全角标点转半角，修复后仍必须过 schema 校验；
6. **校验与重问**：用 zod / pydantic 校验字段类型；失败且允许重试时，把解析错误原文塞回对话让模型自修一轮。

```ts
const PARSERS = [JSON.parse, fromFence, fromTag, balancedScan];
for (const p of PARSERS) {
  const r = tryCatch(p, raw);
  if (r.ok && schema.safeParse(r.data).success) return r.data;
}
throw new ParseError(raw); // 原文入库
```

## 踩坑点

- `\{.*\}` 贪婪匹配会吞掉两个对象之间的文字，非贪婪遇到嵌套就提前截断。括号配对老老实实写状态机，百行以内能搞定；
- 多代码块时取第一个，可能拿到解释文字里的示例。我们的优先级是**标签块 > 围栏块 > 平衡扫描**：约束性 prompt 下标签块最可信；
- jsonrepair 之类修复库会静默做类型猜测（比如字符串数字转数值），修复后必须过 schema，不能裸信；
- 流式场景别边收边解析，等完成标记或 tool_call 结束再走流水线；
- prompt 里 few-shot 示例若带围栏，会反过来强化模型包围栏的倾向，示例与解析层要保持一致。

## 可复用建议

- 收敛成一个 `parseModelJson` 工具函数供插件复用，别各自造轮子；
- 每次失败记录 raw output、命中的解析路径、模型标识。跑一段时间就能看到各路径命中率，这是调整 prompt 约束的数据依据；
- 有 structured output / response_format 能力时优先用，解析层只是兜底，不是主路径；
- 把线上真实失败样本沉淀成回归测试 fixtures，比拍脑袋编的用例有价值得多。

## 总结

对 LLM 输出的正确姿态是"不信任但可恢复"：约束生成、分层解析、校验兜底、失败留痕。解析层做扎实后，上层插件就不必到处写 `typeof x === 'string'` 这类散装防御。这套流水线在我们几个自动化流程里，把解析失败从偶发事故变成了可观测、可回归的指标——工程上的确定性，就是这么一点点攒出来的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/9ac5b8647440293f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/f2b7016a7c76e09c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/45bb6e146005c572.png)

