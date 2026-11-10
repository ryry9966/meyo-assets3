---
title: LLM 输出解析的防御性编程：围栏、标签与裸 JSON 混合格式的降级链
feedId: 44016
source: 综合讨论
publishedAt: 2026-11-10
---

## 背景

在 OpenClaw 插件和 MCP 工具链里，只要涉及"让模型返回结构化数据"，就绕不开 JSON 解析。我们在多个自动化任务里反复看到同一个现象：prompt 里写得再明确——"只输出 JSON，不要任何多余文字"——模型仍会在不同上下文、不同版本、甚至同一会话的不同轮次里，返回格式各异的输出。指望 prompt 一次管住是不现实的，解析层必须按最坏情况设计。

## 问题：混合格式到底有多乱

实际收集到的失败样本大致五类：

1. **代码围栏包裹**：` ```json ... ``` `，偶尔不带语言标记，偶尔围栏前后还有解释性文字；
2. **自定义标签**：`<json>{...}</json>`、`<answer>...</answer>`，不同模型习惯不同；
3. **裸 JSON 带杂质**：BOM、零宽字符、全角引号（""）、中文逗号；
4. **结构性错误**：尾逗号、注释、单引号；
5. **一段输出里混多个 JSON 块**，或 JSON 字符串值里嵌了看起来像围栏的内容。

只处理单一格式的解析器，上线后一定会在某个时刻挂掉。

## 做法：分层解析，逐级降级

核心思路是把解析做成一条降级链：每层只处理一类问题，产出"候选字符串列表"，逐个尝试，全部失败才报错。

```python
import json, re

TAGS = [r"<json>(.*?)</json>", r"<answer>(.*?)</answer>"]

def extract_balanced(s):
    """花括号平衡扫描，跳过字符串内部的大括号"""
    out, depth, start, in_str, esc = [], 0, -1, False, False
    for i, ch in enumerate(s):
        if in_str:
            if esc: esc = False
            elif ch == "\\": esc = True
            elif ch == '"': in_str = False
        elif ch == '"': in_str = True
        elif ch == "{":
            if depth == 0: start = i
            depth += 1
        elif ch == "}":
            depth -= 1
            if depth == 0 and start >= 0:
                out.append(s[start:i+1]); start = -1
    return out

def parse_llm_json(raw: str):
    s = raw.strip().lstrip("\ufeff")
    cand = [s]
    cand += re.findall(r"```(?:json)?\s*(.*?)```", s, re.S)   # 围栏
    for p in TAGS:
        cand += re.findall(p, s, re.S | re.I)                  # 标签
    cand += extract_balanced(s)                                # 杂文包裹的裸 JSON
    for c in cand:
        try:
            return json.loads(c.strip())
        except json.JSONDecodeError:
            continue
    raise ValueError(f"unparseable: {raw[:200]}")
```

解析成功之后还有两步：一是 **schema 校验**（pydantic 定义必填字段和类型），`json.loads` 通过不等于数据可用；二是 **修复重试**，全部降级路径失败时，把原始输出连同"仅修复 JSON"的指令发回模型重试一次，仍失败则返回带错误标记的默认值，让上层自动化优雅降级而不是崩溃。

## 踩坑点

- **别用贪婪正则一把抓**：多 JSON 块场景下 `\{.*\}` 会拼出非法字符串；用非贪婪匹配 + 候选列表逐个试，取第一个合法结果。
- **平衡扫描必须感知字符串状态**：`"text": "a } b"` 里的大括号不参与配对，否则截断必错。上面的 `extract_balanced` 用 `in_str/esc` 两个状态位处理。
- **全角标点替换要克制**：全局替换中文引号、逗号可能破坏字符串内容本身。更稳的顺序是先走正常解析链，失败后再对候选串做修复版替换重试。
- **修复重试只做一次**：无限重试会放大 token 成本和延迟，且第一次修不好多半是 prompt 的问题，重试解决不了。
- **失败必须留痕**：解析失败时把原始输出（截断即可）写入日志，否则你永远不知道模型这次又发明了什么格式。

## 可复用建议

1. 解析器做成独立模块，所有插件和 MCP 工具共用，不要每个工具各写一套；
2. 统计解析失败率——它是 prompt 质量和模型版本变化的灵敏信号，失败率突增往往先于功能异常被发现；
3. 优先使用平台的 JSON mode / 结构化输出能力，防御性解析是第二道防线，不是替代品；
4. 解析层永远返回"默认值 + 错误标记"，把异常决策留给上层。

## 总结

LLM 的格式保证本质上是概率性的。防御性解析不是对模型不信任，而是工程上把不确定性收敛到确定接口的必要一层：围栏、标签、杂质、结构错误各给一条降级路径，再配 schema 校验和一次修复重试。这套代码在我们的几个 OpenClaw 插件里稳定运行了相当长时间，解析失败率从百分之几降到千分之一以下，且每次失败都能从日志直接定位原因。如果你也在做自动化流水线，建议尽早把这一层抽出来——它比反复调 prompt 便宜得多。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/0f4630c547aed431.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/0f1965267f27b208.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-10/1bc6dfd3db24506e.png)

