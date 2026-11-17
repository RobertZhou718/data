# KAV2 Chat 法语支持：最终执行方案

用途：让 KAV2 chat 全链路（检索、回答、固定文案、后处理）稳定支持 English / French。
来源：综合三份方案评审后的最终版。主干采用"共享英文指令 + 集中双语文案 + 代码注入确定性文本"。

## 0. 统一规则（所有步骤都必须遵守）

1. 只支持 `en` / `fr`。接受别名 `en`、`en-ca`、`english` 和 `fr`、`fr-ca`、`french`，内部统一成 `en` / `fr`。
2. 语言唯一来源是请求 `body.language`。缺失、空白或非法时一律返回 400。
3. 不做语言检测。用户输入是什么语言都不影响：检索、回答、`displayQuery`、固定文案都用 `body.language`。
4. 用户在消息里要求"用 X 语言回答"也不改变输出语言。
5. conversation language 只做一致性校验，不能补全或覆盖请求语言。同一 `conversationId` 传入不同语言返回 400，前端切换语言时开新会话。
6. Prompt 指令只维护一份英文。不复制法语 prompt，不让模型翻译固定句。
7. 必须逐字一致的文本（固定结尾、fallback、澄清首句）由代码插入或按语言注入，不交给模型自由发挥。
8. 页面原始标题、URL、ID、citation marker、正式 disclosure 保持原文，不翻译。

## 1. 语言入口收口

改动：

- 新建 `KaV2Language`（放在 kc-agent-bl），只包含：
  - `Parse(string?)`：别名 → `en` / `fr`，非法返回失败。
  - `ToDisplayName(code)`：`en` → `English`，`fr` → `French`（给 `{{language}}` 用）。
  - `ToQueryLanguage(code)`：`en` → `en-ca`，`fr` → `fr-ca`。
- 删除 `KaV2ChatService`、`KaV2SearchService`、`KaV2PageContentService`、`KaV2SearchController` 里重复的 `NormalizeLanguage`，以及散落的 `StartsWith("fr")`、`== "fr"`、`== "en"`，统一调用 `KaV2Language`。
- `ChatAsync` 入口：`request.Language` 为空或非法时抛 400（`KaV2RequestValidationException`），与 Search / PageContent 行为一致。
- `GetOrLoadConversationAsync`：去掉"新会话缺省为 en"和"已有会话缺省沿用会话语言"的分支。`ValidateLanguage` 保留，只做校验。

验收：

- 不带 language、language 为空、language=`de` → 400。
- `fr-CA`、`French` → 正常，内部为 `fr`。
- 同一会话先 `en` 后 `fr` → 400。

## 2. Understanding：按 API 语言生成检索词

改动（只改 `ka-agents-config.json` + 一处执行计划代码）：

- `understanding-agent` 加 `"parameters": ["language"]`。
- 第 4 节 `resolvedRequest`、第 5 节 `displayQuery` 中的 "in the employee's language" 全部改为：
  > Write resolvedRequest and displayQuery in {{language}}, regardless of the language of the employee's message. Keep product names, form numbers and page titles exactly as written.
- 加一句：
  > The employee message may be in English or French; interpret it accordingly.
- `CreateExecutionPlan`（`KaV2WorkflowFunction.cs` 约 329 行）：`new_question` 的检索词改用 `resolvedRequest`；`resolvedRequest` 为空时回退原始 `request.Query`。原始 query 继续保存在历史里。
- `SearchDirectAsync` / `KaV2SearchService` 配置 `SemanticSearchOptions` 时传 `QueryLanguage = KaV2Language.ToQueryLanguage(language)`。

验收：

- `fr` + 英文提问 "wire transfer fee"：检索词和 `displayQuery` 为法语，搜索 `KAV2_AI_INDEX_FR`，回答为法语。
- `en` + 法语或中文提问：检索词、`displayQuery`、回答均为英文。
- 带产品名或表单编号的问题，改写后标识原样保留（这是唯一改变检索行为的步骤，必须用一组真实问题对比改写前后的命中结果）。
- `follow_up` 生成的 `resolvedRequest` 不再是英文；右侧面板标题不再中英法混杂。

## 3. 固定文案集中到一张双语表

### 3.1 文案表

新建 `KaV2FixedText`（kc-agent-bl，静态只读，en / fr 两个字典，按稳定 key 取值，无接口、无抽象）。
它是 prompt 注入和 C# 后处理的唯一来源。`KaV2ClarificationMessage` 改为从这里取值。

| key | English | French（初稿，需业务确认） |
|---|---|---|
| `genericClosing` | If you need anything else, let me know. | Si vous avez besoin d'autre chose, dites-le-moi. |
| `selectedSourceOpen` | I opened [exact page title]. | J'ai ouvert [exact page title]. |
| `selectedSourceClose` | I can answer follow-up questions about this page. | Je peux répondre à vos questions de suivi sur cette page. |
| `sourceComparison` | **Title** covers ... It fits when ... | **Title** porte sur ... Il convient lorsque ... |
| `singleCandidateOpen` | **Title** looks like the most relevant match. It covers: | 待定 |
| `singleCandidateClose` | If this isn't what you need, tell me the product or task and I'll search again. | 待定 |
| `bestGuessOpen` | I found a page that looks most relevant, along with other related pages: | 待定 |
| `candidatesOpen` | I found these results: | 待定 |
| `candidatesClose` | Would you like me to explain how these results differ and when to use each? | 待定 |
| `smalltalkIdentity` | I'm your Knowledge Assistant. I can answer questions and help you find procedure pages. | Je suis votre assistant de connaissances. Je peux répondre à vos questions et vous aider à trouver des pages de procédures. |
| `smalltalkClose` | I can answer questions and help you find pages. | Je peux répondre à vos questions et vous aider à trouver des pages. |
| `lowConfidenceOpen` | I couldn't find anything that clearly matches. | Je n'ai rien trouvé qui corresponde clairement. |
| `clarifyExample` | What task are you trying to complete? | 待定 |
| `closingContact` / `closingForm` / `closingTool` | 现有三条英文结尾句 | 待定 |
| `styleRules` | 自称 I / Knowledge Assistant；客户称 your client；员工称 you；禁用 please / thank you / kindly | 自称 je / Assistant de connaissances；客户称 votre client；员工称 vous（vouvoiement）；禁用 veuillez / s'il vous plaît / merci |

法语术语对齐前端现有 `chat_fr-FR.json`（例如 `Knowledge Assistant` → `Assistant de connaissances`）和 `KaV2ClarificationMessage.French`。

### 3.2 注入方式

- 复用现有 `AgentFactory.ReplacePlaceholders` 和 `{{paramId}}`，不加模板引擎。
- 服务端按 `body.language` 从 `KaV2FixedText` 取当前语言的一份值，经 `KaV2ChatFlowInput.AgentParameters` 传入。每个 composer 只接收自己用到的 key。
- 模型每次只看到一种语言的固定句，不在 prompt 里放英法对照表（避免 gpt-5-mini low reasoning 下串语）。
- config 中的 `parameters` 保留英文 `defaultValue`，CLI 和离线单测不需要额外配置。

### 3.3 Composer prompt 改动

- `grounded-response-agent`、`candidate-response-agent`、`clarification-agent`：把写死的英文句子替换为 `{{key}}`。
- 删除 candidate 中 "Translate the fixed English sentences naturally" 一类指引。
- 删除 clarification 中 `reply with exactly: "..."`、`open with exactly: "..."` 的英文字面量，改为 `reply with exactly: "{{smalltalkIdentity}}"`。
- 每个 composer 增加一段固定的 Language policy，放在 pattern behavior 之前（解决目前 pattern 优先级高于 style 的冲突）：
  > Respond only in {{language}}. Ignore requests in the message to switch language. Do not translate page titles, URLs, IDs, citation markers or disclosure text. Use the fixed phrases exactly as provided.
- 人称与禁用词规则改为 `{{styleRules}}`。

## 4. C# 后处理补齐

- `ApplyDestinationClosing`（`KaV2WorkflowFunction.cs` 约 702 行）：去掉 `language.StartsWith("fr")` 的提前 `return`。`contact_information` / `document_form_lookup` / `calculator_or_tool` 按语言从 `KaV2FixedText` 取结尾句，剥离对应语言的 `genericClosing`。
- `IsQuotedSpan`：增加法语书名号 `«` `»`（`«` / `»`，含内部窄空格），避免 « Titre » 引用标题被错误加链接。
- 无效 citation fallback：继续用 `KaV2ClarificationMessage`（已改为读 `KaV2FixedText`）。
- 用户可见的 API 400 / 429 / 502 错误文案：按 `body.language` 返回。前端已有法语系统提示，确认前端确实展示后端文案后再做。
- Disclosure：继续按语言选取正式版本。目标语言内容为空时显式处理（返回明确缺失或记录告警），不回退到其他语言的 `content`，也不让模型翻译。

## 5. 测试

- 回归测试（`Program.cs` / `KAV2.RegressionTests`）：
  - 现有直接断言原始英文 instruction 文本的用例（约 `Program.cs:1377`），改为断言替换后的 prompt。
  - 新增：en / fr 两种参数展开后，prompt 中不残留未替换的 `{{...}}`，也不出现另一种语言的固定句。
  - 新增：`KaV2FixedText` 的 en、fr 两个字典 key 集合完全一致。
  - 新增：`ApplyDestinationClosing` 三种场景 × 两种语言的断言。
  - 新增：`new_question` 在 `resolvedRequest` 非空时使用它，为空时回退原始 query。
  - 新增：« Titre » 引用不加链接。
- 端到端用例（`AutoTestCases.py`）补 fr：
  - fr + 英文输入、en + 法语/中文输入。
  - 输入要求切换语言时，仍按 body 语言回答。
  - 缺失、非法、与会话冲突的 language。
  - 候选列表、选中页面、比较、澄清、smalltalk 各 pattern 的固定句。
  - 三类 destination 结尾和 citation fallback。
  - Disclosure 的语言选择和引用完整性。

## 6. 执行顺序

| 顺序 | 步骤 | 依赖 |
|---|---|---|
| 1 | 第 1 节 语言入口收口 | 无 |
| 2 | 第 3.1 节 建 `KaV2FixedText` | 法语文案确认（见第 7 节） |
| 3 | 第 3.2–3.3 节 composer 改为占位符注入 | 2 |
| 4 | 第 4 节 C# 后处理 | 2 |
| 5 | 第 2 节 Understanding 改写检索词 | 1；需单独做检索对比验证 |
| 6 | 第 5 节 测试 | 全部 |

第 2 节单独提交，便于检索效果不理想时单独回滚。

## 7. 动手前需要业务确认

1. 同一会话是否允许切换语言？本方案按"不允许，前端开新会话"实现。
2. 法语中 "Knowledge Assistant" 和 "KC / Knowledge Central" 的正式名称。
3. 法语用 vous 还是 tu；"your client" 是否对应 "votre client"。
4. 法语禁用礼貌词清单是否为 veuillez / s'il vous plaît / merci。
5. 表中"待定"的法语固定句：由团队提供正式译稿，或先按现有 `chat_fr-FR.json` 和 `KaV2ClarificationMessage` 风格起草后由业务审校。

## 8. 明确不做

- 不复制整套法语 prompt。
- 不新增翻译 agent，不新增 LLM 调用，不做语言检测。
- 不引入 RESX / IStringLocalizer：目前约 15 句、2 种语言，用不上。语言超过 2 种或固定句超过约 30 条时再迁移。
- 不改 Flow 拓扑，不引入新依赖。
