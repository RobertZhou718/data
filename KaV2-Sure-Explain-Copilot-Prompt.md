# KaV2 "Sure, Explain" — Copilot 实现指令

用途：直接粘贴给 GitHub Copilot，实现 medium / low_intent 候选结果的 "Sure, Explain" 短解释功能。

```text
在 KAV2 中实现 medium/low_intent 候选结果的 "Sure, Explain" 功能。按以下步骤改动，不改 PendingInteraction，不改 API 响应模型。

1. 请求新增 Action
   - Chat 请求模型 → KaV2ChatService → KaV2RequestContext 透传一个可选字段 Action（string?）。
   - 前端点击按钮时发送：{ "query": "<按钮显示文本>", "action": "explain_candidates" }。query 照常作为用户消息存入对话记录，后端判断只看 action，不看文本。

2. KaV2WorkflowFunction.CreateExecutionPlan（第 235 行前）加确定性快路径：
   若 request.Action == "explain_candidates"（定义为常量），且 taskContext.CandidateSets.LastOrDefault() 存在且 Options ≥ 2，
   则把 understanding 改写为：TurnKind=compare_sources，ReferenceStatus=resolved，CandidateSetId=该 set，
   SelectedOptionIds=全部 option.SourceId（按 DisplayIndex），TopicTransition=false，DisplayQuery=""，
   ResolvedRequest = ActiveTask.CurrentGoal ?? ActiveTask.OriginalQuestion ?? request.Query。
   其余流程复用现有逻辑（应得到 SelectedSources + Compare，TaskId 不变，候选集保留）。
   Action 为空或条件不满足时，完全走原逻辑。

3. CreateResponsePolicy：decision 算出后，若 Pattern 为 multiple_candidates_search_results_medium_confidence 或 multiple_candidates_search_results_low_intent，
   且 BuildCandidatePages(providerResult.Evidence.Items.Take(MaximumSelectedResultCount)).Count == 1，
   改为 multiple_candidates_best_guess_high_confidence，contentPanel=source（confidence/reasonCode 不变）。
   需要时把 providerResult 的作用域提到外层。这样 medium/low_intent 返回时一定有 ≥ 2 个候选页，前端据此显示按钮。

4. KaV2FinalResponseRestorer.Restore：让 LinkCandidateTitles 接收按显示顺序排列的 (Title, Url) 序列；
   候选 pattern 照旧传 CandidatePages；source_comparison 额外传 evidence.Items
   （按 KaV2ConfidenceEvaluator.GetStableSourceIdentity 去重，取 Entry.Title / Entry.CibcUrl），使 **title** 变成可点击链接。

5. ka-agents-config.json 的 prompt：
   - candidate-response-agent：替换 medium 与 low_intent 分支，删除 "Which page would you like to view?"：
     "For multiple_candidates_search_results_medium_confidence and multiple_candidates_search_results_low_intent: introduce the list with exactly: "I found these results:" Present every result neutrally as **Exact supplied page title** — one short plain statement of what it covers. Do not call a result a page or a document unless its title does. After the list, end with exactly: "Select a title to open it. Would you like me to explain how these results differ and when to use each?" Never mention buttons or other interface controls."
     best-guess 分支补充："With exactly one candidate page, introduce it with exactly: "I found a page that looks most relevant:" and give its single entry."
   - grounded-response-agent：替换 source_comparison 分支：
     "For source_comparison: stay under 90 words. You may open with one sentence on how the results differ, only if the evidence supports it. Then, for each compared source in the supplied order, write its **exact supplied title** followed by one or two short sentences on when to use it ("Use this for…" / "Use this when…"), choosing the points that matter most for executionPlan.resolvedRequest. Describe differences; never rank or recommend. Never repeat or mention the question. No closing sentence, no offer, no source list, no links. Include at least one resultId per compared source in selectedResultIds; citations = []."
   - understanding-agent（处理用户手打 "yes / what's the difference" 的情况），在 compare_sources 定义后加：
     "When the most recent assistant message (the last entry in recentConversation) offered to explain how the listed results differ and the employee accepts (for example "yes", "sure, explain", or "what's the difference?"), use compare_sources with referenceStatus = resolved, candidateSetId = availableChoices.choiceSetId, and selectedOptionIds = every sourceId in availableChoices. Never treat an earlier offer as pending."

6. 测试（沿用现有测试项目风格）：
   - medium 或 low_intent，且前 3 个 chunk 同属一页 → best-guess，contentPanel=source；
   - action=explain_candidates 且候选集选项 ≥ 2 → SelectedSources + Compare，选中全部选项，TaskId 不变，候选集保留；
   - action=explain_candidates 但候选集不存在或选项 < 2 → 走原逻辑；
   - query="Sure, Explain" 但无 action → 不走快路径；
   - source_comparison 回复中的 **title** 被替换成链接。

7. 更新 kc-agent-bl/KaV2-Flow-Design.md：
   - 请求契约新增 action；
   - Discovery 矩阵 medium/low_intent 行（单页 → best-guess）；
   - Non-discovery policies 的 compare 行（新的解释格式）；
   - 新增 Turn 3 示例：action=explain_candidates → compare。

不在本次范围：用 action 跳过 Understanding 节点（后续性能优化）、清理 PendingInteraction（另开 PR）。

完成后列出改动文件和测试结果。
```

## 前端约定

- 只在 `responsePattern` 为 `multiple_candidates_search_results_medium_confidence` 或 `multiple_candidates_search_results_low_intent` 时显示按钮，且只在最新一条消息上显示。
- 点击发送 `action: "explain_candidates"`；按钮文本跟随 UI 语言，后端不依赖文本。
