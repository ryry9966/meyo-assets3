---
title: LLM 输出解析的防御性编程：一套处理 JSON 标签混合格式的分层方案
feedId: 43679
source: 综合讨论
publishedAt: 2026-11-07
---

## 背景

在 Agent / MCP 插件开发里，让模型输出结构化 JSON 是最常见的集成方式。但管线只要跑得够久就会发现：同一个提示词，在不同模型、不同版本、不同温度下，输出的"包装"五花八门——有时是 ```` ```json ```` 围栏，有时是 `<json>` 标签，有时是裸 JSON 前后还带着一句"好的，以下是结果"。解析器写得太乐观，管线就会在凌晨三点挂掉。

## 问题

真实失败样本大致四类：

- **包装混用**：围栏、标签、裸 JSON 混着来，甚至标签里再套围栏；
- **伪 JSON**：尾逗号、单引号、全角引号和全角冒号（中文模型高发）、Python 风格的 `True/None`、`//` 注释；
- **结构干扰**：JSON 字符串里含围栏或 `}`，把 naive 正则和括号计数全部带偏；
- **多对象**：模型输出两份 JSON，或中途自我纠正了一遍。

## 做法：分层提取 + 校验 + 一次定向重试

核心思路：别指望一个正则解决所有情况，而是按可信度从高到低依次产出候选片段，逐个解析并做 schema 校验，全部失败才报错。

**Step 1：约定单一标签。** 提示词里要求模型用 `<json>...</json>` 包裹输出，这是第一优先级来源，因为它边界最明确。

**Step 2：分层候选提取（顺序固定）：**

```python
BT = "`" * 3  # 避免围栏嵌套
FENCE = re.compile(BT + r"(?:json|JSON)?\s*\n(.*?)" + BT, re.S)
TAG   = re.compile(r"<json>\s*(.*?)\s*</json>", re.S | re.I)

def candidates(text: str):
    for m in TAG.finditer(text):     # 1) 显式标签
        yield m.group(1)
    for m in FENCE.finditer(text):   # 2) 代码围栏
        yield m.group(1)
    yield from balanced_scan(text)   # 3) 裸 JSON：括号配平扫描

def parse(text: str, schema):
    raw = text.strip().lstrip("\ufeff")
    errors = []
    for cand in candidates(raw):
        try:
            obj = json.loads(repair(cand))
            schema.validate(obj)     # 校验失败也算失败
            return obj, None
        except Exception as e:
            errors.append(str(e))
    return None, errors              # 不抛异常，交给上层
```

`balanced_scan` 的关键是带字符串状态做配平：遇到未转义的 `"` 切换 in_string 状态，只在字符串外计数 `{}`。否则字符串里的引号和花括号会把截断位置全弄错。

**Step 3：保守修复。** 只做低风险修补——去 BOM、去尾逗号、把字符串外部的全角引号/冒号映射为半角、归一 `True/False/None`。修完必须过 schema 校验，防止"修出来的错数据"静默进下游。

**Step 4：校验用 jsonschema，保留错误信息**，它是重试的原料。

**Step 5：定向重试，只一次。** 把上次的校验错误拼进提示词重发，明确要求"只输出 JSON"。无限重试只会烧 token，还掩盖真实失败率。最终失败时返回结构化错误对象而非抛异常，让上层自己决定降级策略。

## 踩坑点

- `\{.*\}` 贪婪匹配会把两个对象之间的说明文字一起吃进来；`\{.*?\}` 又会在字符串内部的 `}` 处提前截断。老老实实写配平扫描。
- 全局替换引号去"修"单引号问题，会破坏含撇号的字符串内容。修复只做在字符串边界外。
- 围栏正则遇到"JSON 字符串里包含示例代码"就截断——这也是标签优先于围栏的原因。
- 修复后不校验就放行，比直接报错危害更大。
- 解析失败不落日志，下次还在同一个地方摔。

## 可复用建议

1. 提取器写成纯函数模块，建一个"坏输出语料库"（真实失败原文脱敏后存成 fixture），每次改动跑全量单测，回归一目了然。
2. 有结构化输出 / function calling 就用，解析器是兜底，不是主路径。
3. 失败原文持续入库，语料库会自己长大。
4. 把"解析失败率"当指标记录——格式漂移可以量化，不是玄学。

## 总结

提示词与代码之间的 JSON 约定，本质上是一个会漂移的接口。防御性解析不是把提示词写得更"恳切"，而是承认失约一定会发生：分层提取覆盖形态，保守修复处理方言，schema 校验守住底线，一次定向重试收敛残余。这套东西一个下午能写完，换来的是格式漂移面前，管线不至于整条停摆。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/25c06e106e84a48d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/a6f51f4a74f62d43.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-07/e9f50688f40b711e.png)

