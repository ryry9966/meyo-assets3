---
title: LLM 输出解析的防御性编程：混合 JSON 格式的分层处理管道
feedId: 44229
source: 综合讨论
publishedAt: 2026-11-12
---

## 背景

在 OpenClaw 插件和自动化流程里，很多环节要让模型产出结构化数据：消息分类、字段抽取、MCP 工具的中间结果。提示词里写"只输出 JSON"，听起来是个约定，但实践里它更像一个概率性承诺。我给一个定时批处理任务做过统计：同一份提示词跑两周，约 12% 的返回无法直接 `JSON.parse`，失败集中在深夜无人值守的时段，排查成本很高。

## 问题

失败模式大致五类：

1. **包裹层不固定**：```json 围栏、`<json>` 自定义标签、裸 JSON、"好的，结果如下："前导文本混着来；
2. **字符串值里嵌套**：字段本身含 markdown 代码块或花括号，正则一截就断；
3. **生成中断**：max_tokens 截断，JSON 只剩半截；
4. **字符污染**：全角引号、尾逗号、单引号；
5. **解析成功但结构不对**：数字变字符串、数组变单对象。

## 做法

我把解析拆成一个分层管道，逐层降级：

```js
function parseLLMJson(raw, schema) {
  const text = stripThinkTags(raw);        // 1. 去掉 <think> 等推理块
  if (tryParse(text)) return ok();         // 2. 快路径：直接 parse
  const cands = collectCandidates(text);   // 3. 围栏/标签/括号配对三路候选
  for (const c of ranked(cands)) {         // 4. 按置信度排序尝试
    const r = lenientFix(c);               //    宽松修复：尾逗号、引号
    if (schemaCheck(r)) return ok(r);      // 5. 过 schema 才算成功
  }
  return fail({ raw, repairs });           // 6. 失败也把原文留档
}
```

关键点：

- **提取候选别用"第一个 `{` 到最后一个 `}`"**。写一个感知字符串转义的括号配对扫描，否则会被字段内容里的花括号骗到。
- **围栏正则要考虑嵌套**：字段值里再出现 ``` 时，简单非贪婪匹配会提前截断，配合语言标记能挡住大多数情况。
- **修复后必须过 schema 校验**（zod/ajv）。修复工具有时会"补"出一个合法但语义残缺的对象。
- 全部失败时做一次带错误信息的重试：把上次输出和解析报错回传，让模型自己改格式，重试上限 1~2 次。

## 踩坑点

- **截断修复最危险**：被截断的数组修复后是合法的短数组，校验通过、业务翻车。解析前先看 finish_reason，truncated 直接走重试，不走修复。
- **修复要留痕**：每次 lenientFix 记录改了什么。日志里出现高频修复模式，说明提示词该改，而不是解析器继续兜底。
- **换模型后失败模式会漂移**，老正则可能突然全挂。解析规则做成配置外置，别硬编码在插件里。

## 可复用建议

- 把 `parseLLMJson` 沉淀成插件间公共工具，返回 `{ ok, value, raw, repairs }`，别每个脚本各写一份。
- 提示词侧优先要求模型用自定义标签包裹（如 `<json>...</json>`），比依赖围栏稳定得多；运行时支持 function calling / json_schema 的场景优先用，防御解析只做兜底。
- 攒真实失败样本当单测。我那个项目现在有 47 条用例，每修一个坑就加一条。

## 总结

LLM 的格式约定是概率性的，解析侧就要按概率来设计：先约定格式，再兜住违约的情况。分层管道 + schema 校验 + 原文留档，这三件套让我的批处理失败率从 12% 降到 0.3%，而且每次失败都能在几分钟内定位原因。防御性编程在 LLM 场景不是过度设计，是基本盘。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/3ae693111f567c93.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/b55dbbb42715a8ff.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/1f397034e04969d5.png)

