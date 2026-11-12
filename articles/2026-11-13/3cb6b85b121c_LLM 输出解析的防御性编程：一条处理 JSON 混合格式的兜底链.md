---
title: LLM 输出解析的防御性编程：一条处理 JSON 混合格式的兜底链
feedId: 44275
source: 综合讨论
publishedAt: 2026-11-13
---

## 背景

在 OpenClaw 的插件和自动化流程里，LLM 的结构化输出几乎无处不在：子代理把结果回传主流程、MCP 工具间的中间数据、脚本对模型返回做二次处理。提示词里明明写了“只输出 JSON"，实际拿到的却五花八门：有的被 ```json 围栏包着，有的是裸 JSON，有的前面多一句“以下是结果”，偶尔还混着全角引号和尾逗号。

## 问题

最朴素的做法是 `JSON.parse(resp)`，它只在模型“非常听话”时成立。格式漂移一次，整条自动化链路就断在中途，而且失败往往不稳定复现——同一任务跑五次，四次成功一次炸，排查成本极高。防御性解析的目标不是让模型更听话，而是让解析层对格式漂移免疫。

## 做法：四级兜底链

1. **直解**：原样 `JSON.parse`，成本最低，命中大多数听话场景。
2. **剥围栏**：剥离 \`\`\`json 围栏再 parse，注意按行处理，别用贪婪正则。
3. **括号截取**：定位首个 `{` 或 `[`，做“字符串感知”的括号配对后截取。字符串感知指跳过引号内部的括号，否则值里含 `{` 就会截错位。
4. **修复重试**：针对常见污染做修复——尾逗号、全角引号冒号、单引号键、控制字符，可直接上 `jsonrepair` 这类库；仍失败则带着解析报错回炉给模型，要求“仅输出纯 JSON"。

```ts
export function robustParse(raw: string) {
  const attempts: Array<[string, () => unknown]> = [
    ["direct",  () => JSON.parse(raw)],
    ["fence",   () => JSON.parse(stripFences(raw))],
    ["bracket", () => JSON.parse(sliceBalanced(raw))],
    ["repair",  () => JSON.parse(jsonrepair(raw))],
  ];
  for (const [via, fn] of attempts) {
    try { return { ok: true, data: fn(), via }; } catch { /* 下一级 */ }
  }
  return { ok: false, data: undefined, via: "none" };
}
```

parse 成功 ≠ 语义正确。过完兜底链还要过一遍 schema 校验（zod / jsonschema），字段类型和枚举值对不上照样判失败。

## 踩坑点

- **贪婪正则吞正文**：`\`\`\`.*\`\`\`` 会把两个代码块之间的内容一起吃掉。按行扫描围栏标记更稳。
- **括号配对没做字符串感知**：值里含 `{`、`"` 时截取错位，比不截取更隐蔽，出的是“看似合法的错数据”。
- **过度修复**：单引号转双引号可能误伤本来就含单引号的合法字符串。修复只对直解失败的片段做，修完必须重新 parse + 校验。
- **全角字符**：中文环境下 `“`、`：` 混进 key 和分隔符，肉眼极难发现。日志里把可疑字符的 codepoint 打出来。
- **回炉不带报错**：重试 prompt 里不贴 parser 的具体错误，模型大概率原样再错一遍；贴上错误后一次通过率明显提高。
- **静默使用修复数据**：`via=repair` 命中率偏高，说明提示词该改了，别让解析层替上游背锅。

## 可复用建议

- 全项目只留**一个解析入口**，别在每个插件各写一版，修 bug 时改不齐。
- 返回值带上元信息（命中哪级兜底、原始文本），失败时原始文本落盘，支持回放。
- **统计兜底命中率**：direct 占比过低，先修上游提示词或改用结构化输出通道，而不是继续加解析补丁。
- 拿生产脏数据攒一个“格式语料”测试集，每次改解析器全量跑一遍，防止修好一种弄坏另一种。
- 能用 tool call / JSON mode 这类结构化输出就优先用，防御性解析是长尾兜底，不是主路径。

## 总结

格式漂移是常态，不是异常。把直解、剥围栏、括号截取、修复重试做成有序兜底链，加上 schema 校验和命中率打点，流水线就从“碰运气”变成“可度量”。解析层越钝感，上游的自动化越敢放开手脚。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/ccd3b1fbff0a2ad6.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/cdb45adf8074a955.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-13/68aedd50fb2296d4.png)

