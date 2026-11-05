---
title: LLM 输出解析的防御性编程：混合 JSON 标签格式的分层处理
feedId: 43568
source: 综合讨论
publishedAt: 2026-11-06
---

## 背景

在 OpenClaw 的 agent 工作流和 MCP 工具链里，让模型输出结构化 JSON 是高频操作：插件传参、自动化任务的结果回填、多 agent 数据交接，基本都靠它。Prompt 里写明"只输出 JSON，不要任何多余内容"，实际跑起来模型并不总是听话。

## 问题

归档了一段时间线上失败样本后，脏输出基本是三类：

1. **围栏包裹**：` ```json ... ``` `，语言标注时有时无，偶尔 JSON 字符串字段里还嵌一段 markdown；
2. **标签混用**：`<result>{...}</result>` 这类自定义标签和围栏同时出现，或标签里套围栏；
3. **前后杂质**：开头一句"好的，以下是结果"，结尾跟解释，甚至输出两段 JSON。

下游裸写 `json.loads`，任何一种都会直接炸。更麻烦的是格式分布会随模型、prompt、温度参数漂移——解析层不能写死某一种格式，得做防御性设计。

## 做法：抽取 → 修复 → 校验

把解析当成不可信输入的边界来处理，类似网络 I/O，分三层。

**第一层：抽取**，目标是拿到"最像 JSON 的片段"。优先级：先匹配自定义标签，其次按行首剥离代码围栏，最后用括号配平扫描兜底。注意别用 `\{.*\}` 这种贪婪正则——它会从第一个 `{` 吃到最后一个 `}`，遇到多段 JSON 或字符串含 `{` 必错。配平扫描的核心：

```python
def extract_json(text: str) -> str | None:
    m = re.search(r"<result>(.*?)</result>", text, re.S)
    if m:
        text = m.group(1)
    start = text.find("{")
    if start == -1:
        return None
    depth, in_str, esc = 0, False, False
    for i, ch in enumerate(text[start:], start):
        if esc:
            esc = False
        elif ch == "\\":
            esc = True
        elif ch == '"':
            in_str = not in_str
        elif not in_str:
            if ch == "{":
                depth += 1
            elif ch == "}":
                depth -= 1
                if depth == 0:
                    return text[start : i + 1]
    return None
```

**第二层：修复**，白名单式规则替换：中文智能引号换半角、去尾逗号、单引号键、`True/False/None`、`//` 注释。可以直接用 json-repair 这类库，但每条规则最好对应一种真实观察到的失败模式，别无条件"尽力跑通"。

**第三层：校验**，修复结果必须过 pydantic schema。修复可能"成功"但语义是错的——比如数字被修成字符串——只有 schema 兜得住。校验失败时归档原始输出，再做一次带上下文的修复式重试，上限 1 次，别拖延迟。

整条链路统一返回 `(ok, data, raw, error)`，异常不上抛，调用方按状态分支。

## 踩坑点

- 围栏剥离别全局替换：JSON 字符串字段里可能含 ` ``` `，要按行首匹配外层围栏；
- 智能引号是隐形杀手，中文语境下 `"key"` 肉眼难分辨，报错位置还指不准；
- 贪婪正则匹配多段 JSON 时会拼出一个"看起来完整"的错对象，问题暴露在很下游；
- 修复不能无校验，部分修复库会静默改数据类型；
- 修复式重试的失败率要进监控，突然升高通常意味着上游换了模型或改了 prompt。

## 可复用建议

1. 解析逻辑收敛到一个模块，别在各处散落 `json.loads`；
2. 运行时支持 JSON mode / function calling 就优先用，标签解析只做兜底；
3. 攒一个脏输出语料库当单测 fixture，线上每次解析失败都归档进去，回归测试直接跑它；
4. prompt 里给一个固定格式的 few-shot 示例，能明显降低围栏混用概率，比事后修复便宜。

## 总结

LLM 输出本质是不可信输入，解析层就是它的边界防御：抽取保下限，修复按白名单，校验守语义，重试设上限，再加持续的失败样本归档。这套东西不炫技，但 agent 流水线的稳定性改善非常直接。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/6990809049c84ced.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/17fcb45fd2c8a691.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-06/79b54d413c1827fb.png)

