# KaV2 Sure Explain + R9 — Copilot 实现 review 记录

已看：KaV2WorkflowFunction.cs、KaV2GenericMixedFlowAdapter.cs、ka-agents-config.json、KaV2FinalResponseRestorer.cs、KaV2ChatRequest.cs / KaV2ChatService.cs（用户确认无问题）。
跳过：KaV2ResponseMapper.cs、package*.json（用户之前未提交的改动，与本次无关）、KaV2MixedFlowContracts.cs（只是透传）。
未看：Program.cs（测试）。

## 一、确定要改

1. **Restorer：source_comparison 标题链接可能漏链**
   LinkCandidateTitles 用 cursor 只向后查找，sources 顺序必须与标题在 message 中出现顺序一致。
   source_comparison 传的是 evidence.Items 顺序，composer 按 "supplied order" 输出，两者不一定一致 → 后出现的标题找不到，只打 warning。
   修法：source_comparison 分支每个来源从 message 开头查找；候选列表分支不变。
   测试：比较回复中标题顺序与 evidence.Items 相反时，两个都被链接。

2. **Adapter：override 的 Understanding 只填了 7 个字段**
   Language（以及下游会读的其他字段）缺失 → 法语用户点按钮可能得到英文 / Language 为空。
   修法：Language 从上一轮（ActiveTask 或最近 turn）取；其他下游会读的字段一并补。
   测试：法语会话 action=explain_candidates → Language = fr，回复为法语。

## 二、不确定，需要查代码确认

3. **QuestionForm 优先级被反转**（KaV2WorkflowFunction 464-465、477-480）
   现在 understanding.QuestionForm ?? ActiveTask.OriginalQuestionForm。
   是本次 Copilot 改的，还是用户另一需求（ka-prompt-change-requirements-v2 follow-up 规则）的未提交改动？若是 Copilot 改的 → 撤回。

4. **understanding 新规则依赖 "recentConversation 最后一条是 assistant 的 offer"**
   若 recentConversation 包含本轮用户消息，最后一条永远是用户消息，规则失效 → 需改成 "the most recent assistant message"。
   查 KaV2UnderstandingInputBuilder / BuildUnderstandingPromptInput。

5. **source_comparison 的 citationsRequired**
   grounded prompt 要求 citations = []；若 policy 里 citationsRequired = true，与 Citations 规则冲突，finalize 可能走 fallback。
   查 CreateResponsePolicy 中 source_comparison 的 Policy(...) 参数。

6. **R9 要点素材**
   candidate agent 收到的 candidatePages.supportingChunks 是否带正文（不只是 ID / 标题）？没有则 2–3 条要点无素材。
   查 composer-evidence 构造候选页输入的地方。

7. **override 测试是否走真实 FlowExecutor**
   需覆盖 Understanding 序列化 → 下游反序列化（TurnKind 枚举格式与 LLM 输出一致）；若只直接调 CreateExecutionPlan 则没测到。

8. **37 条构建警告是否有本次新增的**

## 三、小问题（可不改）

- UnderstandingWasOverridden 在 CreateOutput 里重新调用 CreateExplainCandidatesUnderstanding 计算一次，结果一致，只是重复。
- Restorer：GetStableSourceIdentity 若返回空字符串（非 null），多个空身份会被去重成一个；用户确认 CibcUrl 可靠，暂不处理。

## 四、真实 LLM 验收（修完后手动跑）

| 场景 | 预期 |
|---|---|
| medium 多页结果 | "I found these results:" 开头，固定结尾问句，无 "Which page…" |
| 接着 action=explain_candidates | understandingCalls=0；< 90 词；标题都是链接；"Use this when…"；不推荐不排名 |
| 不带 action，手打 "yes" | 走 understanding，结果也是比较 |
| 隔一轮别的问题再打 "explain" | 不解释之前那组候选 |
| 单页 medium / low_intent | R9：标题 + 2–3 条要点 + 固定结尾，无引用，要点无编造 |
| 普通 "compare A and B" | 同样的新比较格式 |
| 法语跑列表、按钮比较、R9 | 固定文案翻译正确，Language = fr |
