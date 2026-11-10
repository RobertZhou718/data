# Citation 后端修复 — Copilot 实现指令

用途：直接粘贴给 GitHub Copilot。本次只改后端，不改前端，不改 prompt。

```text
按你上一轮的核对结论，实现下面 4 项后端改动。不改前端，不改 ka-agents-config.json。

业务规则（不变）：要求 citation 的直接回答，最终至少要有 1 条有效 citation，否则返回固定模板回复。

1. 降级为模板回复时，contentPanel.mode 改为 "search_results"
   位置：MapChat 里 noAnswer 分支（policy.CitationsRequired && assignedCitations.Count == 0 && !suppressedOnPurpose）。
   - message 换成模板、来源清空的现有行为保持不变。
   - 同时把 contentPanel.mode 设为 "search_results"，defaultSourceId / defaultCitationId 为 null。
   - responsePattern、resultsConfidence 保持不变。
   - 确认写入 StoredTurn / SQL 的 ContentPanelMode 用的是改后的值。
   - 确认这种情况下响应里仍然带搜索结果列表，面板有内容可显示；如果没有，先告诉我，不要自行补。

2. 降级日志带原因
   - 现有的降级 Warning 日志加上本轮 citation 失败原因的汇总（复用已有的 CitationDiagnostics，
     按原因码计数，例如 no_html=2, low_match=1）。
   - LCS 超过 MaxLcsCells 时使用独立原因码 lcs_cell_limit，并记录 source / markdown 的 token 数，
     不再和真正的 low_match 混在一起。
   - 只加日志，不加新的指标设施，不改变任何判定结果。

3. 模型漏写 selectedResultIds 时自动补齐
   - citation.resultId 只要属于服务端的 selectedEvidence.Items，就不因为模型没把它列进
     selectedResultIds 而丢弃。
   - 继续做 unit、范围和 locator 的全部现有校验。
   - 通过校验的 citation.resultId 合并回最终的 selectedResultIds，让 Mapper 使用同一份结果。
   - 补齐时记录独立原因 selected_result_inferred。
   - 未知的、或不属于 selectedEvidence.Items 的 resultId 仍然拒绝（原因码不变）。

4. FrontendQuoteLocator.Create 里把 pageText.ToString() 移到循环外，只执行一次。

测试（沿用现有 citation 回归测试项目的风格）：
   - 直接回答且有效 citation 为 0 → message 为模板，contentPanel.mode = "search_results"，
     默认来源 / 默认引用为 null，responsePattern 不变，持久化的 mode 一致。
   - 有效 citation ≥ 1 → 行为和现在完全一样。
   - 有意抑制（fullPageCitationThreshold）的情况不受影响。
   - 模型引用了 selectedEvidence 内的 resultId 但没列进 selectedResultIds → citation 保留，
     最终 selectedResultIds 包含它。
   - 模型引用了 selectedEvidence 之外的 resultId → 仍被丢弃。
   - LCS 超限 → 原因码为 lcs_cell_limit。
   - 现有 103 个回归测试全部通过。

不在本次范围：
   - PDF 页码（前端目前不使用 pageNumber）。
   - 前端 matcher 的重复文本消歧和非 BMP 字符偏移问题。
   - 面板内容索引与引用证据索引是否同一份内容。
   - 表格行匹配补分规则。
   - LCS 对齐缓存。

完成后列出改动文件和测试结果。
```
