---
title: LLM 输出解析的防御性编程：一套应对 JSON 混合格式的分层方案
feedId: 43933
source: 综合讨论
publishedAt: 2026-11-09
---

## 背景

在 OpenClaw 的插件、Agent 流水线和 MCP 工具链里，让 LLM 输出 JSON 是最常见的集成方式：工具参数、结构化抽取、中间结果传递。但只要接的模型多、任务杂，很快会遇到同一个现实：你要求“只输出 JSON”，模型却给你各种花样——

- ` ```json ` 代码围栏包裹
- 前后带一句“以下是解析结果：”
- `<think>` 推理标签之后才跟 JSON
- 自定义标签如 `<tool_call>{...}</tool_call>`
- 尾逗号、单引号、中文引号、注释
- `max_tokens` 截断产生的半截 JSON

单模型时代可以针对一种格式写死解析。现在多模型混用、版本频繁更替，**格式漂移是常态而不是异常**，解析层必须是防御性的。

## 问题

两类典型故障：

1. 直接 `json.loads(resp)`，平时能跑，偶发崩溃，且崩溃时拿不到原始输出，无法复盘。
2. 用宽松正则提取后“解析成功”，但内容根本不符合预期 schema，错误在下游才爆出来，排查成本更高。

## 做法：一条分层提取管道

核心思路：**不要赌格式，按成本从低到高逐层尝试，每层留下痕迹。**

```python
import json, re

def extract_json(text: str):
    # L1 直接解析
    try:
        return json.loads(text), "direct"
    except Exception:
        pass
    # L2 代码围栏
    m = re.search(r"```(?:json|json5|jsonc)?\s*(.*?)```", text, re.S)
    if m:
        try:
            return json.loads(m.group(1)), "fence"
        except Exception:
            text = m.group(1)
    # L3 括号平衡扫描（跳过字符串内的大括号）
    start = min((i for i in (text.find("{"), text.find("[")) if i != -1), default=-1)
    if start != -1:
        depth, in_str, esc = 0, False, False
        oc, cc = text[start], ("}" if text[start] == "{" else "]")
        for i in range(start, len(text)):
            c = text[i]
            if in_str:
                if esc: esc = False
                elif c == "\\": esc = True
                elif c == '"': in_str = False
            elif c == '"': in_str = True
            elif c == oc: depth += 1
            elif c == cc:
                depth -= 1
                if depth == 0:
                    return json.loads(text[start:i+1]), "brace"
    raise ValueError("no json found")
```

后续步骤：

1. **剥包装**：先去掉 `<think>`、自定义标签等已知外壳。
2. **修复兜底**：L1–L3 都失败时，交给 `json_repair` 这类库处理尾逗号、单引号、智能引号。
3. **Schema 校验**：用 pydantic / jsonschema 校验结构和字段类型。解析成功不等于结果正确。
4. **带反馈重试**：校验失败时，把错误信息拼回 prompt 重试一次，设硬预算（比如 2 次），超了就大声失败，不要静默吞掉。

## 踩坑点

- **正则贪婪匹配**：`re.S` 下用 `.*` 会跨块吞掉多个 JSON 对象，务必非贪婪或限定围栏。
- **平衡扫描忘了字符串内的大括号**：`{"tpl": "{a}"}` 这类数据会让朴素计数器提前归零，必须维护 `in_str` 状态。
- **截断 JSON 修不回来**：repair 能修尾逗号，修不了缺失字段。先看 `finish_reason` 是否截断，是就直接重请求。
- **`{}` 和 `[]` 也能解析成功**：schema 校验里把空结构、必填字段一起卡掉。
- **没有留底**：所有策略都失败时，如果没存原始输出，你只能干瞪眼。原始响应必须进日志。

## 可复用建议

- 把 extract + repair + validate 封装成一个公共工具函数，放在插件/Agent 的共享库里，别在每个插件里各写一份。
- 记录每次命中了哪一层策略，形成简单的统计，模型格式漂移会最先体现在这个指标上。
- 模型侧能用 structured output / JSON mode 就优先用，解析层只做兜底，而不是第一道防线。
- Prompt 里仍然要求“仅输出 JSON，不要围栏”，但永远不要只靠 prompt。

## 总结

LLM 的输出本质上是不受控的外部输入，解析边界应该按处理不可信数据的标准来做：分层降级、逐层留痕、结构校验、失败要响。这套管道代码量不到五十行，但能把“偶发崩溃、查无实据”变成“有日志、有降级、有指标”的常规工程问题。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/02f912138f89ee83.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/144234624d5c6399.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-09/9c1d07dac6b5c1c2.png)

