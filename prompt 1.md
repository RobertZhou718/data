# KaV2 "Sure, Explain" + R9 — Copilot 实现指令

用途：直接粘贴给 GitHub Copilot。本次只改后端。

```text
在 KAV2 中实现两件事：(A) medium/low_intent 候选结果的 "Sure, Explain" 短解释；(B) R9 单候选 best-guess 的新格式。
只改后端；不改 API 响应模型；不新增或修改 PendingInteraction。

1. 请求新增 Action
   - Chat 请求模型 → KaV2ChatService → KaV2ChatFlowInput → adapter 透传可选字段 Action（string?）。
   - 前端点击按钮发送：{ "query": "<按钮显示文本>", "action": "explain_candidates" }。
     query 照常作为用户消息存入对话记录；后端只看 action，不看文本。

2. 确定性路由（用现有 agent override 生成 Understanding，不在 CreateExecutionPlan 局部改写）
   条件：request.Action == "explain_candidates"（定义常量）
         且 taskContext.ActiveTask != null
         且 taskContext.CandidateSets.LastOrDefault() 存在且 Options ≥ 2。
   满足时 override 直接返回 Understanding：
     TurnKind=compare_sources，ReferenceStatus=resolved，CandidateSetId=该 set，
     SelectedOptionIds=全部 option.SourceId（按 DisplayIndex），TopicTransition=false，DisplayQuery=""，
     ResolvedRequest = ActiveTask.CurrentGoal ?? ActiveTask.OriginalQuestion ?? request.Query。
   override 不调用 understanding LLM；相应修正 understandingCalls 日志（此时为 0）。
   之后复用现有流程：SelectedSources + Compare，TaskId 不变，候选集保留。
   条件不满足时完全走原逻辑（不做过期/消息绑定校验，由前端在下一轮让按钮失效）。

3. 单候选改道（CreateResponsePolicy）
   decision 算出后，若 Pattern 为 multiple_candidates_search_results_medium_confidence
   或 multiple_candidates_search_results_low_intent，
   且 BuildCandidatePages(providerResult.Evidence.Items.Take(MaximumSelectedResultCount)).Count == 1，
   改为 multiple_candidates_best_guess_high_confidence，contentPanel=source（confidence/reasonCode 不变）。
   需要时把 providerResult 的作用域提到外层。
   结果：medium/low_intent 返回时一定有 ≥ 2 个候选页，前端据此显示按钮。

4. 标题链接（KaV2FinalResponseRestorer.Restore）
   让 LinkCandidateTitles 接收按显示顺序排列的 (Title, Url) 序列；
   候选 pattern 照旧传 CandidatePages；
   source_comparison 额外传 evidence.Items（按 KaV2ConfidenceEvaluator.GetStableSourceIdentity 去重，
   取 Entry.Title / Entry.CibcUrl），使 **title** 变成可点击链接。CibcUrl 可依赖，现有 fallback 保留。

5. ka-agents-config.json 的 prompt
   - candidate-response-agent，best-guess 分支（R9），替换为：
     "For multiple_candidates_best_guess_high_confidence: with exactly one candidate page, write exactly "**Exact supplied title** looks like the most relevant match. It covers:" followed by two or three short bullets taken only from that candidate's supporting content, then end with exactly: "If this isn't what you need, tell me the product or task and I'll search again." Never mention other pages. No citations. With two or more candidate pages, keep the current intro and list unchanged."
   - candidate-response-agent，medium 与 low_intent 分支，替换为（删除 "Which page would you like to view?"）：
     "For multiple_candidates_search_results_medium_confidence and multiple_candidates_search_results_low_intent: introduce the list with exactly: "I found these results:" Present every result neutrally as **Exact supplied page title** — one short plain statement of what it covers. Do not call a result a page or a document unless its title does. After the list, end with exactly: "Select a title to open it. Would you like me to explain how these results differ and when to use each?" Never mention buttons or other interface controls."
   - grounded-response-agent，source_comparison 分支，替换为（普通比较与 "Sure, Explain" 共用这一种行为）：
     "For source_comparison: stay under 90 words. You may open with one sentence on how the results differ, only if the evidence supports it. Then, for each compared source in the supplied order, write its **exact supplied title** followed by one or two short sentences on when to use it ("Use this for…" / "Use this when…"), choosing the points that matter most for executionPlan.resolvedRequest. Describe differences; never rank or recommend. Never repeat or mention the question. No closing sentence, no offer, no source list, no links. Include at least one resultId per compared source in selectedResultIds; citations = []."
     90 词是 prompt 软约束，不做代码校验或截断。
   - understanding-agent，在 compare_sources 定义后加（处理用户手打 "yes / what's the difference"）：
     "When the most recent assistant message (the last entry in recentConversation) offered to explain how the listed results differ and the employee accepts (for example "yes", "sure, explain", or "what's the difference?"), use compare_sources with referenceStatus = resolved, candidateSetId = availableChoices.choiceSetId, and selectedOptionIds = every sourceId in availableChoices. Never treat an earlier offer as pending."
   - 法语：沿用现有行为，固定英文文案由模型翻译。

6. 测试（沿用现有测试项目风格；同步更新受影响的 prompt 测试旧文本断言）
   - medium 或 low_intent，且前 3 个 chunk 同属一页 → best-guess，contentPanel=source，confidence 不变；
   - best-guess 单候选：输出为 标题 + 2–3 条要点 + 固定结尾，无引用，不提其他页面；多候选输出不变；
   - action=explain_candidates，ActiveTask 存在且选项 ≥ 2 → 不调用 understanding LLM，SelectedSources + Compare，
     选中全部选项，TaskId 不变，候选集保留；
   - action=explain_candidates 但 ActiveTask 为空、候选集不存在或选项 < 2 → 走原逻辑；
   - query="Sure, Explain" 但无 action → 不走快路径；
   - source_comparison 回复中的 **title** 被替换成链接。

7. 更新 kc-agent-bl/KaV2-Flow-Design.md
   - 请求契约新增 action；
   - Discovery 矩阵 medium/low_intent 行：单页 → best-guess；
   - best-guess 行：R9 单候选格式；
   - Non-discovery policies 的 compare 行：新的解释格式（普通比较同样适用）；
   - 新增 Turn 3 示例：action=explain_candidates → compare（override 跳过 understanding LLM）。

不在本次范围：后端 action 过期/消息绑定校验、新增或修改 PendingInteraction（另开 PR 清理）、前端改动。

完成后列出改动文件和测试结果。
```

## 前端约定（不在本次实现范围）

- 只在 `responsePattern` 为 `multiple_candidates_search_results_medium_confidence` 或 `multiple_candidates_search_results_low_intent` 时显示按钮，且只在最新一条消息上显示；用户未点击就进入下一轮时，按钮失效。
- 点击发送 `action: "explain_candidates"`；按钮文本跟随 UI 语言，后端不依赖文本。
