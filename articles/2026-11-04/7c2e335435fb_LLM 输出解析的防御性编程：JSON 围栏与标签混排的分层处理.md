---
title: LLM 输出解析的防御性编程：JSON 围栏与标签混排的分层处理
feedId: 43414
source: 综合讨论
publishedAt: 2026-11-04
---

## 背景

只要 Agent 链路里有一环是"让模型返回结构化数据"，就迟早会撞上同一个问题：prompt 里明明写了"用 ```json 围栏输出"，实际拿到的却五花八门。函数调用和结构化输出并不是处处可用——本地小模型、多厂商路由、模板跨版本演进、MCP 工具结果里内嵌的 LLM 文本，都会让输出格式失去保证。容错解析不是可选项，是基础设施。

## 问题

同一个"返回 JSON"的指令，线上真实出现的形态至少有这些：

- 标准围栏 ` ```json ... ``` `；
- 裸 JSON，前后带解释性文字；
- 围栏语言标注大小写/拼写混乱（`JSON`、`javascript`）；
- `<json>` / `<result>` 之类的标签包裹，和围栏混用；
- 尾逗号、中文引号、字符串内换行；
- 最阴险的一种：JSON 字段值里本身含 ` ``` `，嵌套围栏。

任何只写一条正则的方案，都会在其中两三种形态上翻车。

## 做法

思路是分层提取、宽进严出。每一层失败就落到下一层，最后由括号配平扫描兜底：

```python
import json, re

FENCE = re.compile(r"```[a-zA-Z]*\s*(.*?)```", re.S)
TAG   = re.compile(r"<(?:json|output|result)>\s*(.*?)\s*</(?:json|output|result)>", re.S)

def candidates(text: str):
    yield text.strip()
    yield from (m.group(1) for m in FENCE.finditer(text))
    yield from (m.group(1) for m in TAG.finditer(text))

def json_block(s: str):
    """取第一个括号配平的 JSON 块，字符串内的括号不计数。"""
    depth, start, in_str, esc = 0, -1, False, False
    for i, ch in enumerate(s):
        if start < 0:
            if ch in "{[": start, depth = i, 1
            continue
        if in_str:
            if esc: esc = False
            elif ch == "\\": esc = True
            elif ch == '"': in_str = False
        elif ch == '"': in_str = True
        elif ch in "{[": depth += 1
        elif ch in "}]":
            depth -= 1
            if depth == 0: return s[start:i + 1]
    return None

def parse_llm_json(text: str):
    for cand in candidates(text):
        block = json_block(cand) or cand
        try:
            return json.loads(block)
        except json.JSONDecodeError:
            continue
    raise ValueError("unparseable")  # 记录原文，触发带错误信息的重试
```

括号扫描是关键兜底：即使嵌套围栏把正则候选截断了，它也能从原文里抠出第一个配平的完整对象。语法修复（尾逗号、中文引号）可以用 `json_repair` 这类库作为再下一层，解析成功后再过 pydantic/jsonschema 校验。

## 踩坑点

- **裸括号计数**：不区分字符串状态的 depth 计数，遇到 `"示例": "输出 { 和 }"` 直接错位。必须带字符串与转义状态机。
- **贪婪正则**：`re.S` 下用 `.*` 会把两个对象吞成一个。非贪婪 + 配平扫描，取"第一个完整块"；多个对象的取舍策略（取首个还是报错）要显式定义。
- **静默修复掩盖漂移**：修复库很方便，但如果删尾逗号、改引号这些动作不留日志，模型行为退化你根本发现不了。修复率本身应该是个监控指标。
- **重试无上限**：解析失败就"再生成一次"，在坏模板上会空转。重试必须带具体报错（截断、尾逗号），上限 1–2 次，再失败升级到 function calling。
- **流式截断**：边流边 parse 必然遇到半截 JSON。要么缓冲到围栏闭合，要么用增量解析器，别在 on_chunk 里硬解。
- **大小写与空白**：` ```JSON `、围栏后带换行——`[a-zA-Z]*\s*` 这种宽容写法成本几乎为零。

## 可复用建议

1. 收敛成一个共享工具函数，全链路复用，禁止每个插件各写一份正则。
2. 把线上真实失败样本脱敏后沉淀为 fixture 跑回归——这是这类代码唯一可靠的测试方式。
3. 分层顺序固定：原文 → 围栏 → 标签 → 括号扫描 → 语法修复 → schema 校验，并记录"在哪一层成功"，据此对比不同模型、不同模板版本的漂移。
4. prompt 侧配合：明确"只输出 JSON，不要任何解释"，提取类任务 temperature 降到 0。但永远不要只靠 prompt。

## 总结

防御性解析的本质，是把"模型输出的不确定性"当作常态输入来设计：多策略提取是宽进，schema 校验是严出，日志与指标贯穿全程。围栏写法会变、模型会换代，但这套分层套路基本可以原样搬进你现有的 Agent 工程里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/284e5db33fd672de.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/337775308b05fdf7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-04/1c02172c984f7f09.png)

