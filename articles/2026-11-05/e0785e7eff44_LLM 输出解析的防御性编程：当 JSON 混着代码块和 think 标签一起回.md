---
title: LLM 输出解析的防御性编程：当 JSON 混着代码块和 think 标签一起回来
feedId: 43507
source: 综合讨论
publishedAt: 2026-11-05
---

## 背景

在 OpenClaw 的插件和自动化脚本里，让模型返回 JSON 是高频需求：MCP 工具分派参数、定时任务的决策结果、插件间的结构化消息。提示词里明明写了"只输出 JSON，不要 markdown"，真实运行中仍会遇到各种变体：围栏代码块包裹、前后带解释性散文、推理模型的 `<think>` 段落、长输出被 max_tokens 截断。单次失败率看着只有百分之几，但一个 agent 循环一天跑几百次调用，就是每天都有人报"偶发解析失败"。

## 问题

多数解析代码按理想情况写：`json.loads(text)` 一把梭。它的问题不只是抛异常，而是**静默失败**——异常被吞掉后走了默认分支，agent 执行了错误的动作，排查时日志里只有一行 ValueError。

## 做法：把解析做成一条分层管线

核心思路：解析不是一次尝试，而是一串**有序降级的策略**。每层失败才进下一层，并记录命中了哪一层。

1. **源头优先**：模型/网关支持 tool calling 或 JSON schema 模式时，用它。解析兜底是给不支持的场景准备的，不是首选。
2. **预处理**：去 BOM 和零宽字符；剥离 `<think>...</think>`；多个代码块时先记下所有候选。
3. **顺序尝试**（骨架如下）：
   - 直接 `json.loads`；
   - 正则抽取所有 ``` 围栏块，逐个尝试；
   - 括号配平扫描：找第一个配平的 `{...}` 或 `[...]`，扫描时跳过字符串字面量内的括号和转义引号；
   - 修复层（尾逗号、全角引号）只作最后手段，修复后必须再校验，并打日志。

```python
def extract_json(text: str):
    t = strip_noise(text)                      # BOM、think 段
    try:
        return json.loads(t, parse_constant=_reject), "direct"
    except JSONDecodeError:
        pass
    cands = re.findall(r"```(?:json)?\s*(.*?)```", t, re.S)
    cands.append(scan_balanced(t))             # 括号配平，感知字符串
    for c in cands:
        try:
            return json.loads(c, parse_constant=_reject), "extract"
        except JSONDecodeError:
            continue
    log.warning("json repaired, raw=%s", t[:200])
    return json.loads(repair_json(t), parse_constant=_reject), "repair"
```

4. **Schema 校验 + 一次性反馈重试**：校验失败时，把具体错误（缺哪个字段、哪个类型不符）拼回提示词重试一次。不要无限重试同一提示词。
5. **可观测**：统计各层命中率。如果 extract 层占比持续上升，说明该改提示词了，而不是继续加固解析器。

## 踩坑点

- `\{.*\}` 贪婪匹配会吞到后文的散文；非贪婪 `\{.*?\}` 又死在嵌套括号上。老老实实写字符感知的配平扫描，十几行代码的事。
- 多个围栏块时，第一个可能是提示词示例的回显。**逐个尝试**，取能通过校验的那个，而不是取第一个。
- `json.loads` 默认接受 `NaN`/`Infinity`，这不是合法 JSON，用 `parse_constant` 显式拒绝。
- 修复库对截断 JSON"修得很好"，但语义可能是错的，数字和布尔值都可能被改。截断应该走"加预算重试"，不走修复。
- 别在解析器里做业务默认值兜底。缺字段就报错重试，否则错误会沉到三层之下才爆，更难查。

## 可复用建议

- 收敛成一个工具模块：`extract_json(text, schema)`，返回 `(data, strategy, warnings)`，业务代码不接触原始文本。
- 把线上真实的失败样本攒成回归语料，解析器任何改动必须过这批用例。
- 提示词侧同步减方差：JSON 契约放在提示词**末尾**、少夹带无关指令，解析层的压力会小很多。

## 总结

防御性解析是安全带，不是底盘。正确的顺序是：生成时约束（结构化输出）→ 解析时容忍（分层降级）→ 事后校验（schema）→ 持续观测（分层指标）。解析器写得越来越强，往往说明上游契约没谈好；把它控制在"最后一层兜底"的体量，agent 的行为才可预测。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/fe55ab96c889a119.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/c11e0eedb90e1fc4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-05/974016e0a1cba2d7.png)

