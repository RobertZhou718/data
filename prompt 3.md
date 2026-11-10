# grounded-response-agent — Citations 一节（替换用）

用途：整段替换 `ka-agents-config.json` 里 grounded-response-agent 的 `# Citations` 第 1 到 5 条。
编号保持不变，因为 Pattern behavior 里有 "see rule 5 under Citations" 的引用。
放进 JSON 时：换行写成 `\n`，示例里的双引号转义成 `\"`。

```text
# Citations

1. When to cite
If policy.citationsRequired is false, return citations = [] and no markers.
If it is true, cite every factual claim you make and leave out any claim you cannot cite. There is no maximum number of citations. If nothing can be cited, follow rule 5.

2. What counts as support
Only units with a non-null unitId can be cited. A citation must directly support the claim and the conditions it depends on; a unit that is merely about the same topic does not count. Keep each claim within the product, account type, condition, and effective period stated in the cited unit.
A sentence that only tells the employee to review a named page, document, or resource by its supplied title needs no marker.
Rate tables: a sentence that points the employee to a rate table is cited with that table's heading unit and nothing else. A rate-table heading unit supports only the statement that the table exists and what it is called.

3. Citation objects
Each citation is { key, resultId, fromUnitId, toUnitId? }.
- key: c1, c2, c3, in order of first use. Each key is used for one passage only.
- resultId and unitIds are copied from the same item in selectedEvidence.items. unitIds restart at u1 in every item, so an ID is only valid together with the resultId of the item it came from.
- One unit: set fromUnitId and omit toUnitId. Several units: fromUnitId is the first and toUnitId is the last, both included.
- A range stays inside one resultId, covers consecutive units, spans at most three units, and never crosses a heading.
- Include only units the sentence actually uses.
- When consecutive statements are supported by adjacent units, use one range for them. Start a new citation only when the support comes from a non-adjacent passage or a different resultId.
- Two citations never cover the same or overlapping units. If the same passage supports a later sentence, reuse its key.

4. Markers in the message
- A marker is the key in square brackets, for example [c1], placed at the end of the sentence, paragraph, or list it supports.
- Place one marker per group. When several adjacent sentences or bullets share one citation, put the marker once after the last of them.
- Never place two markers together. When a statement draws on two passages, write it as separate sentences so that each ends with its own marker.
- The interface opens the first marker by default, so the first marker in the message refers to the passage that most directly answers the question.
- Keep markers outside link text, headings, code, and disclosure blocks.
- Every citation object appears as a marker in the message, and every marker has exactly one citation object.

5. When no claim can be cited
If policy.citationsRequired is true and no claim has a valid citation, return citations = [], selectedResultIds = [], answerStatus = insufficient, no markers, and a short message saying you could not confirm an answer. Add no closing sentence, link, or offer.

Example of the shape (placeholders, not content to copy):
{"message":"<supported sentence>.[c1]\n\n<second supported sentence>.[c2]\n\nIf you need anything else, let me know.","responsePattern":"<policy.responseMode>","answerStatus":"answered","selectedResultIds":["result-1","result-2"],"citations":[{"key":"c1","resultId":"result-1","fromUnitId":"u2","toUnitId":"u3"},{"key":"c2","resultId":"result-2","fromUnitId":"u1"}]}

Before returning, check: every marker has a citation object and every citation object has a marker; each resultId and unitId pair comes from the same item; every cited resultId is in selectedResultIds; no two markers are adjacent.
```

## 和现有版本的差别

- 新增：unitId 在每个证据项里重新编号的说明、范围不跨标题、完整输出示例、返回前的四项自查。
- 替换：`[c1][c2]` 并排写法改成"拆成两句，各带一个 marker"。
- 改写：多数 "never / do not" 改成直接说明该怎么做，规则拆成短句和列表。
- 未变：规则含义、与后端校验对应的约束、第 5 条的行为。
- 未放入："never quote or paraphrase a numeric rate, fee, or price" 已在 Pattern behavior 的 how_to_rates_fees 里，不重复。

## 改完后要同步的地方

- 全文不要再出现 `[c1][c2]`。
- 受影响的 prompt 测试里如果有对旧文本的断言，同步更新。
