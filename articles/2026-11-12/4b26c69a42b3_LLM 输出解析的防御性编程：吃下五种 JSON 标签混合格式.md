---
title: LLM 输出解析的防御性编程：吃下五种 JSON 标签混合格式
feedId: 44195
source: 综合讨论
publishedAt: 2026-11-12
---

## 背景

在 Agent 流水线里，我们经常让 LLM 输出 JSON：意图识别结果、MCP 工具调用参数、字段抽取。Prompt 里写死"只输出 JSON，用 ```json 包裹"，测试集上很稳，一上线就漂移：有时带围栏，有时不带；有时套标签，有时标签内外各包一层；偶尔还附带一句"以下是结果"。这不是某个模型的问题，而是所有模型在不同上下文长度、采样温度下的共同行为。解析层必须按"输入永远是脏的"来设计。

## 问题：同一套 prompt，至少五种输出形态

线上日志里真实见过：

1. 纯 JSON：`{"a": 1}`
2. 围栏包裹：\`\`\`json ... \`\`\`
3. 标签包裹：`<json>{...}</json>`
4. 双重包裹：`<result>` 里面再套一层 ```json 围栏（嵌套格式）
5. 前后带说明文字：`好的，结果如下：{...}`

只处理其中一种的解析器，一旦失配就是下游故障——MCP 工具收到非法参数，插件直接抛异常。

## 做法：多候选提取 + 分层解析

核心思路：不赌单一格式，把文本里所有"可能是 JSON 的片段"都提出来，按优先级逐个尝试。用 `JSONDecoder.raw_decode` 而不是裸 `json.loads`——它能容忍前后杂文本，且正确处理字符串值内部的 `{}`（手写括号计数器在这里必翻车）。

```python
import re
from json import JSONDecoder

FENCE = re.compile(r"```(?:json)?\s*(.*?)```", re.S | re.I)
TAG   = re.compile(r"<\s*(\w+)\s*>(.*?)<\s*/\s*\1\s*>", re.S | re.I)
KNOWN = {"json", "result", "answer", "tool_call", "output"}
dec = JSONDecoder()

def parse_llm_json(text: str):
    cands  = [text]
    cands += [m.group(1) for m in FENCE.finditer(text)]
    cands += [m.group(2) for m in TAG.finditer(text)
              if m.group(1).lower() in KNOWN]
    for c in cands:
        c = c.strip().lstrip("\ufeff")
        starts = [0] + [i for i, ch in enumerate(c) if ch in "{["][:5]
        for s in starts:                    # 容忍前后杂文本
            try:
                obj, _ = dec.raw_decode(c[s:])
                return obj                  # raw_decode 正确处理字符串内的 {}
            except ValueError:
                continue
    raise ParseError(text[:200])            # 带原始片段抛结构化错误
```

解析成功后不要直接返回：用 pydantic 或 jsonschema 做 schema 校验，失败时带截断的原始输出抛结构化错误。连续失败可带错误信息重试一次，明确告诉模型"你上次带了 markdown 围栏，请只输出 JSON"，比笼统重发有效得多。

## 踩坑点

- **贪婪正则**：`<.*>(.*)</.*>` 不加 `re.S` 会漏换行，加了又会跨块吞内容，这是此类 bug 的最高频来源。
- **括号计数**：手写 `{}` 平衡计数，遇到字符串值里的 `}` 直接错位，务必用 `raw_decode`。
- **修复库慎用**：`json_repair` 这类库能把残缺 JSON"修"出来，但也可能静默丢字段、改语义。修复结果必须再过 schema 校验，不能当全信。
- **标签与围栏变体**：模型会写 `<JSON>`、`< json >`、漏闭合标签，甚至只写半个围栏。匹配时做大小写归一、允许空格；闭合缺失时退化为"从开标签/开围栏截到结尾"。
- **重试别只重发原 prompt**：把具体格式错误喂回去，否则同一种失败重复 N 次，白烧 token。
- **温度 0 也不保格式**：推理后端切换、模型小版本升级都可能改变输出习惯，解析层兜底不能省。

## 可复用建议

- 提取逻辑收敛成一个模块、一个入口函数，返回 `{ok, data, strategy, raw_snippet}`，下游统一消费，别在每个插件里各写一套正则。
- 沉淀"格式语料库"：线上解析失败时把原始输出（脱敏、截断）存进 fixture，变成单元测试用例。这个语料库比任何 prompt 技巧都保值。
- 记录哪条解析路径命中最多：如果长期靠兜底路径才成功，说明该改 prompt 了，而不是继续叠正则。
- 有结构化输出能力（JSON mode / function calling / schema 约束）时优先用，手写解析是没有这些能力时的防线，不是替代品。

## 总结

LLM 的格式遵循是概率性的，prompt 能降低漂移概率，但降不到零。防御性解析就三条：多候选提取、分优先级尝试、解析后强制 schema 校验。再把失败输出变成测试语料，让解析层随线上数据持续变硬，而不是每次出事都临时加一个 if。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/0dfbb76c0bbf70d2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/ffc78d91bed199fb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/8142aaf36b674572.png)

