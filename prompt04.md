# KAV2 Chat 英法双语 — 后端实现计划

用途：在后端让 KAV2 chat 的回复稳定支持 English 和 French。
范围：只改后端（`ka-agents-config.json`、配置加载、`KaV2ChatService`、后处理）。前端不动。

## 前提（不要改变）

- 语言由前端通过 API 的 `language` 参数传入，同一会话内语言固定，中途变更返回 400。
- 后端不做 query 语言检测。英文会话里用法语提问，仍然查英文索引、用英文回复。
- 所有 prompt 指令保持英文、只维护一份。不要为法语复制一份 prompt。
- 固定句式不让模型翻译，由后端按语言注入。

## 设计

`parameters` 是声明（有哪些占位符），新增的顶层 `locales` 是数据（每种语言的值）。
运行时按会话语言取出一个 locale 块，并入现有的参数字典，prompt 里照常用 `{{key}}`。

```json
"parameters": [
  { "id": "language", "description": "The language the agent should respond in", "defaultValue": "English", "required": false },
  { "id": "locale", "description": "Locale key used to select a block from locales", "defaultValue": "en", "required": false },
  { "id": "clarificationOpener", "description": "Fixed opening sentence of a clarification reply", "required": true },
  { "id": "candidatesIntro", "description": "Fixed sentence introducing the candidate page list", "required": true },
  { "id": "closingLine", "description": "Fixed closing sentence of a grounded reply", "required": true },
  { "id": "styleRules", "description": "Person, tone, and banned-word rules for this language", "required": true },
  { "id": "maxWords", "description": "Word limit for a reply in this language", "required": true },
  { "id": "lexicalHints", "description": "Language-specific cue words for intent classification", "required": true }
],
"locales": {
  "en": {
    "clarificationOpener": "I couldn't find anything that clearly matches.",
    "candidatesIntro": "I found a page that looks most relevant, along with other related pages:",
    "closingLine": "If you need anything else, let me know.",
    "styleRules": "Refer to \"your client\", never address the client as \"you\". Do not use \"please\", \"thank you\", or \"kindly\". Do not call yourself \"Knowledge Assistant\".",
    "maxWords": "90",
    "lexicalHints": "Pricing cues: fee, rate, pricing. Document cues: form. Affirmative replies: yes, sure, explain."
  },
  "fr": {
    "clarificationOpener": "Je n'ai rien trouvé qui corresponde clairement.",
    "candidatesIntro": "J'ai trouvé une page qui semble la plus pertinente, ainsi que d'autres pages connexes :",
    "closingLine": "Si vous avez besoin d'autre chose, dites-le-moi.",
    "styleRules": "Écrivez « votre client », ne vous adressez jamais au client par « vous ». N'utilisez pas « s'il vous plaît », « veuillez » ni « merci ». Ne vous présentez pas comme « Knowledge Assistant ».",
    "maxWords": "110",
    "lexicalHints": "Pricing cues: frais, taux, tarification. Document cues: formulaire. Affirmative replies: oui, expliquez-moi."
  }
}
```

法语文案里 `clarificationOpener` 和 `closingLine` 取自代码里已有的文本（`KaV2ClarificationMessage`、grounded-response-agent 的结束语）。其余法语句子是初稿，上线前需要法语母语者确认。

## 步骤

### 1. 配置模型和加载

- 在 `AgentsConfiguration` 上加 `Locales`，类型为 `Dictionary<string, Dictionary<string, string>>`，默认空字典。现有加载器会忽略未知字段，所以不加这个属性时 `locales` 会被静默丢弃。
- 在 `AgentFactory.ResolveParameters` 里，解析出 `locale` 的值后，把 `config.Locales[locale]` 的键值并入结果。优先级：运行时传入的值 > locale 块 > `defaultValue`。
- 在 `ConfigurationLoader.Validate` 里加两条检查：
  - 每个 locale 块的键集合必须相同。
  - locale 块里的每个键都必须在 `parameters` 里声明过。

注意：现在 `required: true` 的参数没有值时不会报错，占位符会原样留在 prompt 里（`ResolveParameters` 的现有行为），`Validate` 的错误也只是记 Warning。所以"缺翻译"要靠第 6 步的测试拦住，不要依赖 `required`。

### 2. KaV2ChatService 传入 locale

构造参数字典的两处（聊天流程和 `LoadConfigurationForStatelessAgents`）各加一项：

```csharp
["locale"] = language.SearchLanguage,   // "en" 或 "fr"
```

`language`、`roleFilter`、`searchIndexName` 保持不变。

### 3. 把 prompt 里的硬编码英文换成占位符

对 `ka-agents-config.json` 里的每个 composer agent：

- **clarification-agent**：`Open with exactly: "I couldn't find anything that clearly matches."` 改成 `Open with exactly: "{{clarificationOpener}}"`。这同时解决了 `# Priority` 规定 pattern behavior 优先于 style、而 pattern behavior 写死英文的冲突。
- **candidate-response-agent**：`multiple_candidates_best_guess_high_confidence` 分支写死的 `"I found a page that looks most relevant, along with other related pages:"` 改成 `{{candidatesIntro}}`。另外两个分支里 `Translate the fixed English sentences naturally` 这句删掉，把它们各自的固定句也搬进 `locales`，一句一个键。
- **grounded-response-agent**：结束语改成 `{{closingLine}}`。`source_comparison` 里的 `"Use this for..."` 句式同样搬进 `locales`。
- **三个 composer 共用**：人称、禁用礼貌词、自称的规则换成 `{{styleRules}}`；`90 words` 换成 `{{maxWords}} words`。
- 每个 agent 的 `"parameters"` 数组里加上它用到的键。占位符只对列在这个数组里的参数生效，漏列就不会被替换。

改完后全文搜索，prompt 里不应再有任何带引号的英文固定句。

### 4. understanding-agent 接收会话语言

- `"parameters"` 从 `[]` 改成 `["language", "lexicalHints"]`。
- 第 4、5 条的 `in the employee's language` 改成 `in {{language}}`。这样法语会话里输入 `GIC`、`TFSA`、`form 80`、`1` 这类短词或缩写时，`resolvedRequest`（用于检索 `KAV2_AI_INDEX_FR`）和 `displayQuery` 仍然是法语。
- 第 2、7 条写死的英文线索词（`fee, rate, pricing, or form word`、`yes` / `sure, explain`）换成 `{{lexicalHints}}`。
- `BuildUnderstandingPromptInput` 生成的输入 JSON 不需要再加 `language` 字段，指令里已经有了。

### 5. 后处理正则按语言分支

`CleanInternalResponseDetails`（`KaV2WorkflowFunction.cs`）里的 `[ \t]+([,.;:!?])` 会删掉所有标点前的空格，法语里 `:` 前的空格是正确排版，会被改坏，并且误触发 `Removed internal identifiers from the KAV2 composer message before returning it.` 这条 Warning。

- 法语会话只对 `,` 和 `.` 生效：`[ \t]+([,.])`。英文保持原样。
- 把会话语言传进这个方法（它现在拿不到语言的话，从 request 上下文取）。
- 确认只有在真的移除了内部标识符时才打那条 Warning，标点清理不应触发它。

### 6. 测试

- **配置测试**：加载 `ka-agents-config.json`，断言 `en` 和 `fr` 两个 locale 块键集合一致；分别用 `locale=en` 和 `locale=fr` 解析后，所有 agent 的 instructions 里不含 `{{`。
- **后处理测试**：法语输入 `Il couvre : A, B` 经过 `CleanInternalResponseDetails` 后保持不变，且不产生 Warning。
- **法语回归用例**：每个 response pattern 至少一条。断言回复以对应的法语固定句开头或结尾，且不包含任何英文固定句。
- **短输入用例**：法语会话输入 `GIC`，断言 `resolvedRequest` 走的是法语索引、`displayQuery` 不是英文。

## 不在这次范围内

- 前端硬编码的英文文案（`View results`、`aria-label`、`Collapse`、`ResultsView.tsx`）。
- 根据 query 自动切换语言或索引。
- 新增第三种语言。结构已经支持：加一个 `locales` 块，并在 `NormalizeLanguage` 和 `GetSearchIndexName` 里加对应分支。

## 顺手确认

`KAV2_AI_INDEX_FR` 的可搜索文本字段用的是法语 analyzer（例如 `fr.microsoft`）。如果用的是默认 analyzer，法语词形变化的召回会差，这是索引侧的事，和上面的改动互不依赖。
