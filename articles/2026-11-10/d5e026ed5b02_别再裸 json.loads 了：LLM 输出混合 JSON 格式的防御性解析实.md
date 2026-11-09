---
title: 别再裸 json.loads 了：LLM 输出混合 JSON 格式的防御性解析实践
feedId: 43949
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景

在 OpenClaw 的插件和 Agent 工作流里，LLM 输出经常被当作结构化数据直接消费：MCP 工具调用参数、自动化任务的中间状态、插件间的消息传递。但 LLM 的输出格式从来不是稳定契约——同一个 prompt，今天返回裸 JSON，明天包一层 ``` 围栏，后天在 JSON 前后多两句解释性中文。解析层一脆，整条链路就挂。

## 问题

真实场景里的"混合格式"至少有五类：

1. **围栏包裹**：```json ... ```，有时连语言标注都没有；
2. **前后缀噪声**：模型在 JSON 前写"好的，结果如下："，后面补一句"如需调整请告诉我"；
3. **多对象**：一次吐出两个 JSON，或对象与数组混着来；
4. **伪合法 JSON**：尾逗号、单引号、中文引号、未转义换行；
5. **思考标签污染**：`<think>` 段落里也有一段长得像 JSON 的内容，朴素正则会抓错。

我们在一条日均 200 次调用的自动化流水线上统计过，裸 `json.loads` 失败率约 7%，全部来自以上几类。

## 做法：分层剥离 + 逐级降级

核心思路：不写"万能正则"，而是写一条有次序的清洗管线，每层只解决一类问题，全部失败再降级。

- **第 1 层·预清洗**：剥 BOM、按标签对剥掉 `<think>` 段、去围栏。找围栏用"首个 ``` 之后、下一个 ``` 之前"，防代码内容本身含围栏。
- **第 2 层·括号配对定位**：不要用贪婪正则抓 `{.*}`，改为从第一个 `{` 或 `[` 做一次考虑字符串转义的深度计数，找到配对闭合点。天然规避"字符串里含 }"和"JSON 后跟解释文字"两类问题。
- **第 3 层·宽容解析**：标准 loads 失败后交给 json-repair 类容错库处理尾逗号、单引号、中文引号。它可能"猜"，所以猜完必须过校验。
- **第 4 层·校验与降级**：产物必须过 pydantic / JSON Schema；失败则带 schema 报错重试（prompt 加"只输出 JSON"，并把上次原始输出作为反例 few-shot）；仍失败就落盘原文 + 告警，绝不静默吞掉。

骨架大致是：

```python
def extract_json(text: str):
    text = strip_think_tags(text).strip()
    text = strip_code_fence(text)
    for cand in balanced_candidates(text):
        try:
            return validate(json.loads(cand))
        except Exception:
            continue
    repaired = validate(json_repair.loads(text))  # 兜底
    raise ParseError(raw=text)
```

## 踩坑点

- `re.search(r'\{.*\}', text, re.S)` 遇到"两个 JSON + 中间文字"会抓出混合体，还把正确的第一个对象弄丢；
- 剥 `<think>` 只找闭合标签时，未闭合输出会切错位置——找不到闭合就整段标记可疑，交给降级层；
- json-repair 可能改写字段值，关键字段要加长度/枚举约束再放行；
- 流式输出别解析半截 JSON，等 done 信号或用增量括号计数确认完整；
- 日志务必保存原始输出原文，否则排障只能靠猜。

## 可复用建议

1. 解析收敛到单一模块，所有 Agent/插件统一引用，不要各自手搓；
2. 解析失败率做成指标上报，突增通常意味着换模型或改了 prompt，是很好的回归信号；
3. 能用原生 function calling / 结构化输出的场景，就不要靠 prompt 约定 JSON，稳定性差距很大；
4. 重试时把 schema 报错回传给模型，比一句"请重新输出"有效得多。

## 总结

LLM 输出不是 API 契约，而是自然语言的副产品。防御性解析的本质是：分层剥离、逐级降级、强制校验、失败可观测。这套管线在社区几个自动化项目里跑了几个月，解析失败率从 7% 降到 0.3% 以下，剩下的基本都是真正的模型异常。代码不到百行，值得每个做 Agent/插件的人写一次。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/edf5ec8fe9c71a52.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/cad9544d6c112d29.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/09365e634a2e345e.png)

