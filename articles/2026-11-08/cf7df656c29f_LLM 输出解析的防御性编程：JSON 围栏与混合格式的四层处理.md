---
title: LLM 输出解析的防御性编程：JSON 围栏与混合格式的四层处理
feedId: 43812
source: 综合讨论
publishedAt: 2026-11-08
---

## 背景

在 OpenClaw 插件和 MCP 工具链里，让模型「输出 JSON」是最常见的约定：工具参数要回填、Agent 步骤间要传结构化数据。但实际拿到的输出从来不是干净的 JSON——有的包 ` ```json ` 围栏，有的裸奔，有的前后加一句「以下是解析结果」，有的写成 ` ```JSON ` 大写，还有嵌套两层围栏的。直接 `json.loads()` 十次挂两三次，自动化任务半夜静默失败，日志里只有一屏 Traceback。

## 问题

表面是格式问题，本质是工程问题：解析逻辑散落在各个插件里，每处只处理自己见过的那几种格式，没有统一入口，也没有回归保障。正确的做法是承认「模型输出是不可靠 I/O」，建一个独立的防御性解析层。

## 做法

分层处理，每层只做一件事：

**1. 提取层：** 先剥代码围栏（兼容大小写和无语言标注）；没有围栏就做括号配对扫描——从首个 `{` 或 `[` 开始计数，跳过字符串内部的花括号，取到配对结束为止。别用贪婪正则，嵌套对象必翻车：

```python
import re

def extract_json(text: str) -> str:
    text = text.strip()
    m = re.match(r"^```[a-zA-Z0-9_-]*\s*\n(.*)\n```\s*$", text, re.S | re.I)
    if m:
        text = m.group(1).strip()
    for opener, closer in (("{", "}"), ("[", "]")):
        start = text.find(opener)
        if start == -1:
            continue
        depth, in_str, esc = 0, False, False
        for i in range(start, len(text)):
            ch = text[i]
            if in_str:
                if esc:
                    esc = False
                elif ch == "\\":
                    esc = True
                elif ch == '"':
                    in_str = False
            elif ch == '"':
                in_str = True
            elif ch == opener:
                depth += 1
            elif ch == closer:
                depth -= 1
                if depth == 0:
                    return text[start:i + 1]
    raise ValueError("no balanced JSON block")
```

**2. 修复层：** 针对尾逗号、单引号、全角引号、`NaN/Infinity` 这类「接近 JSON 但不是 JSON」的内容做有限修复。先试原始 `json.loads`，能过就不动，修复是兜底不是默认路径。

**3. 校验层：** 用 pydantic 或 jsonschema 校验结构和字段类型。这步不能省——语法对了字段照样可能是错的，类型和枚举值必须在边界上拦住。

**4. 重试层：** 校验失败时，把解析报错和原始输出一起回传给模型：「你的输出解析失败，原因是 X，请重新输出」。带错误信息的重试，成功率远高于盲目重试。

## 踩坑点

- 围栏语言标注可能是 `json5`、`javascript`，甚至模型幻觉出来的词，正则别写死 `json`。
- `json.loads` 默认接受 `NaN/Infinity`，不是严格 JSON，下游序列化时才炸，极难排查。
- 字符串里含未转义引号或 ` ``` ` 时，朴素计数会失效，扫描必须维护 `in_str`/`esc` 状态。
- 流式输出别边收边 parse，攒完再解析；确要增量就换支持不完整 JSON 的解析器。
- 修复必须留痕：记录原始输出和实际走的路径（直取 / 剥围栏 / 修复），否则坏数据会静默上线。

## 可复用建议

- 解析层做成独立模块，全社区插件共用一份，别各写各的。
- 用真实运行中的失败样本攒一个「坏输出语料库」，做成单元测试——这个测试集比任何文档都管用。
- 有结构化输出 / JSON mode / 工具调用就优先用，但兜底解析照样保留，接第三方模型或老版本时你会感谢它。
- schema 变更带版本号，旧输出的兼容窗口提前定好。

## 总结

围栏提取、有限修复、schema 校验、带错误回传的重试，四层各司其职，才可能把「偶尔成功」的 demo 变成可运维的自动化。这套东西不性感，但省下的是凌晨翻日志的时间。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/003cc76eb665e72b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/503e68981e856a41.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-08/1d4a30e5a887f9ac.png)

