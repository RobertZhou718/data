# Citation 后端逻辑 — review 核对清单

用途：直接粘贴给 GitHub Copilot。先只核对、不改代码；每项回答"是否成立 + 代码位置 + 建议改法"。
依据：Copilot 之前给出的 citation 后端说明和代码摘录，未看完整源码，所以每项都需要对照代码确认。

```text
请对照当前工作区代码逐项核对下面的问题。先不要改代码。
每项回答：是否成立、对应代码位置、如果成立建议怎么改（最小改动）。

一、高优先级

1. 技术性失败被当成"没找到内容"
   现状：要求 citation 的回答最终有效 citation 数为 0 时，MapChat 把答案替换成固定澄清问题
   （"I couldn't find anything that clearly matches…"）并清空来源。
   问题：citation 失效的原因很多是技术性的，不是内容没找到：
     - HTML 为空（no_html）
     - LCS 超过 MaxLcsCells，返回空映射
     - 匹配率不到 90%（low_match）
     - locator_not_first
     - 模型引用了合法证据但漏写 selectedResultIds
   用户按提示换个说法再问，命中同一篇文档，还会同样失败。
   请核对：
     a. 每种失败原因现在有没有带原因码的日志或指标？没有的话列出需要加的位置。
     b. 答案被替换成澄清问题后，responsePattern、contentPanel.mode、answerStatus
        和写入会话存储 / active-conversation 状态的字段有没有一起改？
        摘录里只看到改了 message 和 assignedSources。
     c. 下一轮 understanding 看到的上一轮状态是"已回答"还是"澄清"？

2. PDF 页码提示可能偏差
   现状：pageNumber 从 contentId 尾部的 _pN 解析。抓取时 Trunk 包含前后重叠页（示例配置前后各 2）。
   问题：_p4 这个 chunk 的文字可能实际在第 2 到 6 页，pageNumber=4 不一定对。
   请核对：
     a. 运行环境里实际的重叠页配置值。
     b. 索引文本里有没有页边界标记可以用来算出真实页码。
     c. 前端在第 N 页找不到 exact 时的行为（有没有找相邻页或全文）。

3. HTML locator 的前后端约定
   现状：后端验证的是"exact 在当前 section + 本轮已选的同源更早 sections 里是第一次匹配"，
   不是整页唯一；不够时把 exact 向后、向前扩展，最多 200 字符。
   PDF 用的是另一套规则：prefix + exact + suffix 在 chunk 纯文本里唯一。
   问题：前端加载的是整页。同一句话如果在未被检索到的更早章节出现过，可能高亮错位置。
   请核对：
     a. 前端对 HTML 和 PDF 分别是怎么用 exact / prefix / suffix / headingText 定位的。
     b. 后端跳过的内容（script、style、隐藏内容、非当前语言的 disclosure 分支）
        和空白归一化，前端是否做了同样处理。
     c. HTML citation 的 headingText 实际会不会有值（示例响应里是 null）。
     d. 前端定位失败时的表现（不高亮、滚到标题、还是报错）。
     e. 内容面板加载的 HTML 是索引里存的那份，还是实时页面？两者不一致时 locator 会失效。

二、需要查证

4. selectedResultIds 漏写导致 citation 被丢弃
   Finalize 要求 citation.resultId 必须在模型返回的 selectedResultIds 里。
   如果 resultId 本身属于选中证据、只是模型没列出来，是否可以由后端自动补上而不是丢弃？

5. 90% 门槛的放宽规则
   BuildUnits 里，没按位置对上的 token 只要出现在同一区域的 MatchTokenValues 里就计入 mappedCount；
   大表格的行参考的是整张表的词集合。
   请核对：这是否可能让费率表里的 unit 被归到错误的行；tokenCount 为 0 的 unit 是否会被当成可引用。

6. LCS 的内存和耗时
   AlignTokens 每个证据块分配 int[source+1, markdown+1]，上限 1600 万格（约 64MB）。
   请核对：每个请求会对多少个证据块做对齐、有没有缓存、超过上限时有没有日志。
   如果没有缓存，评估按 chunk 内容哈希缓存对齐结果是否可行。

7. 多轮 citation 编号
   Finalize 里 marker 写成 [^citation-{finalIndex + 1}]，从 1 开始。
   "同一问题的后续轮次继续编号"的偏移是在哪里加的？message 里的 marker 和 citation.Id 是否保证同步？
   另外：存入会话历史的 assistant 消息里的 [^citation-N] 在传给下一轮 LLM 之前有没有去掉？

8. FrontendQuoteLocator.Create 的扩展预算
   循环里的判断是 nextEnd - sectionEnd + targetStart - extendedStart > MaxExactExtensionChars。
   请确认这里各变量的坐标系一致（section 内偏移还是页内偏移），预算确实是 200 字符。
   另外 pageText.ToString() 在循环内每次都调用，确认是否可以提到循环外。

三、已知限制（不用改，确认描述准确即可）

- 后端不做语义验证；单条 citation 失败只删 marker，保留句子。
- CanonicalizeHtmlRange 在范围超过 3 个 region 或跨标题时截断，而不是拒绝。
- 以 rate-table 标题开头的范围只保留标题本身。
- 扩展后的 exact 可能比模型引用的多出最多 200 字符。
- fullPageCitationThreshold 的比例按 chunk 计算而不是整页，默认关闭。

完成后按上面的编号列出结论。
```
