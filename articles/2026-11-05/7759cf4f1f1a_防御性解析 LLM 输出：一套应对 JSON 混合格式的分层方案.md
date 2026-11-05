---
title: 防御性解析 LLM 输出：一套应对 JSON 混合格式的分层方案
feedId: 43508
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

在 OpenClaw 里写 skill 或插件时，最常见的模式是：让模型按约定输出 JSON，下游代码解析后驱动后续动作。Prompt 里写上"只输出 JSON，不要任何多余内容"，听起来就够了——但只要换模型、升级模型版本，或者中间串了一层推理模型，这套约定就会碎。

## 问题：格式从来不是一种

真实输出常见的形态：

- 裸 JSON（理想情况）
- ` ```json ` 围栏包裹
- `<think>...</think>` 之后才给结果（推理模型几乎必现）
- 先说一句"好的，以下是结果："再贴 JSON
- 中文语境下的全角引号、全角逗号、尾逗号
- 流式截断导致的半截 JSON

`json.loads()` 只认第一种。于是出现典型的"本地测试好好的，上线就炸"，且换一个模型就换一种炸法。

## 做法：分层剥离，逐级降级

核心思路：不写万能正则，而是写一条流水线，每层只处理一类污染，逐层尝试。

```python
import json, re

def extract_json(text: str):
    # 1. 剥离 think 块（含未闭合的情况）
    text = re.sub(r"<think>.*?(</think>|$)", "", text, flags=re.S)
    # 2. 收集候选：围栏内容优先，原文兜底
    fences = re.findall(r"```(?:json)?\s*(.*?)```", text, flags=re.S)
    for c in fences + [text.strip()]:
        try:
            return json.loads(c)          # 3. 直接解析
        except json.JSONDecodeError:
            pass
        seg = _slice_json(c)              # 4. 字符串感知的大括号定位
        if seg:
            try:
                return json.loads(seg)
            except json.JSONDecodeError:
                pass
        try:
            return json.loads(_repair(c)) # 5. 轻量修复后重试
        except json.JSONDecodeError:
            continue
    raise ValueError("no parsable JSON")
```

两个关键细节：`_slice_json` 不能单纯数大括号，要先跳过字符串字面量再计数，否则 `{"s":"a}b"}` 会切错；`_repair` 只做确定性高的替换——全角标点转半角、去尾逗号，不要试图修复一切，修不了就抛错。

最后一层是 schema 校验（pydantic 或 jsonschema）。JSON 合法不等于结构正确，`{"action":"search"}` 漂移成 `{"actions":[...]}` 这种字段级问题，只有 schema 能拦住。

## 踩坑点

1. **贪心匹配围栏**：输出里有多个代码块时（先给示例再给结果），正则会切错位置。取"最后一个完整围栏"通常更稳。
2. **`<think>` 未闭合**：流式截断时常见，所以正则用 `$` 兜底。
3. **指望 temperature=0 或"prompt 里说清楚了"**：格式漂移来自模型版本迭代，不是采样参数。
4. **盲目上 json_repair 类库**：能救一部分，但可能改错语义（数字精度、字段名），修复后仍要过 schema。
5. **解析失败重发同样的 prompt**：大概率复现同样的格式。要把解析错误喂回去："上次输出无法解析，错误是 X，请重新输出"。
6. **不留证据**：失败时把原始输出完整落日志，否则线上只能靠猜。

## 可复用建议

- 解析逻辑收敛到一个工具模块，全项目唯一入口，别在每个 skill 里各写一份正则。
- 记录每条输出"用了哪层策略解析成功"，这个统计能告诉你该换模型还是该改 prompt。
- 有 function calling / JSON mode 就优先用，防御解析器做兜底而不是替代。
- 把线上真实的失败输出脱敏后存成测试夹具，每次改解析器都跑一遍——这是整套方案里回报最高的习惯。

## 总结

LLM 的输出格式是外部输入，不是合同。防御性解析的本质：假设任何约定都可能被打破，用分层降级代替单一正则，用 schema 兜住语义，用日志和测试夹具把故障变成可迭代的资产。这套东西不性感，但它决定了你的 agent 是"偶尔抽风没法查"，还是"随时可修可回归"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/da5d39abcbea50fd.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/ad2ee9fffaf4411d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/9ec00d219e2524f8.png)

