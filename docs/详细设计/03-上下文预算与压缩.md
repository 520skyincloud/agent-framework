# 上下文预算与压缩

## 设计目标

- 在每次模型调用前判断继续、warning、自动压缩、完整压缩、降级压缩或阻塞。
- 用确定阈值处理 200k、1M、20k 模型切换。
- 让实现者知道每个数字的含义、单位、触发动作和调参后果。
- 提供可直接使用的压缩提示词和失败恢复分支。

## 非目标

- 不负责长期记忆排序。
- 不负责业务模型选择。
- 不在 20k 模型里保留完整历史。

## 核心规则

- 默认上下文窗口是 `200_000 tokens`。
- 大上下文窗口是 `1_000_000 tokens`。
- 普通大模型的 compact 输出 reserve 是 `20_000 tokens`。
- 20k 小模型的 reserve 改为 `min(modelMaxOutputTokens, 4_000)`。
- 自动压缩缓冲是 `13_000 tokens`。
- 阻塞前最后保留空间是 `3_000 tokens`。
- warning/error 提前量是 `20_000 tokens`。
- 自动压缩连续失败 `3` 次后熔断。
- `promptTokens > blockAt` 时禁止直接调用模型。

## 状态机

~~~mermaid
stateDiagram-v2
  state "估算 prompt tokens" as Estimate
  state "计算有效窗口" as ComputeWindow
  state "安全继续" as Continue
  state "发出 warning" as Warn
  state "自动压缩" as AutoCompact
  state "完整压缩" as FullCompact
  state "降级压缩" as DowngradeCompact
  state "更激进压缩" as AggressiveCompact
  state "只保留任务状态" as TaskStateOnly
  state "阻塞" as Block

  [*] --> Estimate
  Estimate --> ComputeWindow
  ComputeWindow --> Continue: prompt <= warningThreshold
  ComputeWindow --> Warn: warningThreshold < prompt <= autoCompactAt
  ComputeWindow --> AutoCompact: autoCompactAt < prompt <= blockAt
  ComputeWindow --> FullCompact: prompt > blockAt
  ComputeWindow --> DowngradeCompact: 目标模型窗口变小
  DowngradeCompact --> AggressiveCompact: 第 1 次仍超限
  AggressiveCompact --> TaskStateOnly: 第 2 次仍超限
  TaskStateOnly --> Block: 第 3 次仍超限
  Continue --> [*]
  Warn --> [*]
  AutoCompact --> [*]
  FullCompact --> [*]
  Block --> [*]
~~~

## 数据结构

~~~ts
type ContextBudgetInput = {
  promptTokens: number
  contextWindow: number
  modelMaxOutputTokens: number
  compactFailures: number
  isDowngrade: boolean
  targetModel: string
}

type ContextBudgetDecision =
  | { action: "continue"; promptTokens: number; warningThreshold: number; autoCompactAt: number; blockAt: number }
  | { action: "warn"; promptTokens: number; warningThreshold: number }
  | { action: "auto_compact"; promptTokens: number; autoCompactAt: number }
  | { action: "full_compact"; promptTokens: number; blockAt: number }
  | { action: "downgrade_compact"; promptTokens: number; targetWindow: number; targetModel: string }
  | { action: "block"; promptTokens: number; reason: string }

type CompactSummary = {
  goal: string
  latestUserIntent: string
  currentState: string
  keyFiles: string[]
  openTasks: string[]
  toolResultSummaries: string[]
  risks: string[]
  irreversibleSideEffects: string[]
}
~~~

## 默认值

| 参数名 | 中文含义 | 单位 | 默认值 | 为什么是这个值 | 触发行为 | 调大后果 | 调小后果 |
|---|---|---|---:|---|---|---|---|
| `defaultContextWindow` | 默认模型上下文窗口 | tokens | `200_000` | 200k 能覆盖大多数编程任务，又不假设所有 provider 都支持 1M。 | 未显式指定模型窗口时使用。 | 长历史更少压缩，但成本和延迟更高。 | 长任务更容易提前 compact。 |
| `largeContextWindow` | 大上下文模型窗口 | tokens | `1_000_000` | 给长仓库、多 Agent 汇总和超长 transcript 使用。 | 路由到大上下文模型时使用。 | 可保留更多历史，但调用更慢更贵。 | 大会话更早进入压缩。 |
| `compactOutputReserve` | 大模型输出/摘要预留上限 | tokens | `20_000` | 20k 足够输出完整压缩摘要和长回答，同时不会无限侵占 prompt 空间。 | 计算 `effectiveWindow` 时扣除。 | 输出更安全，但 prompt 可用空间减少。 | prompt 空间更大，但输出截断风险上升。 |
| `smallModelReserveCap` | 20k 小模型输出预留上限 | tokens | `4_000` | 小模型总窗口只有 20k，继续预留 20k 会让 prompt 空间归零。 | 目标模型 `contextWindow <= 20_000` 时使用。 | 小模型输出更充分，但可输入内容更少。 | 输出更容易截断，模型可能无法完整说明恢复步骤。 |
| `autoCompactBuffer` | 自动压缩缓冲 | tokens | `13_000` | 给 token 估算误差、工具 schema 和 provider 包装留安全余量。 | prompt 超过 `autoCompactAt` 时触发自动 compact。 | 更早压缩，安全但频繁。 | 更晚压缩，provider 拒绝风险上升。 |
| `blockingReserve` | 阻塞前最后保留空间 | tokens | `3_000` | 给错误解释、恢复消息和 provider 包装留下最后空间。 | prompt 超过 `blockAt` 时禁止直接调用模型。 | block 更早触发，用户更常需要 compact 或换模型。 | block 更晚触发，provider 返回 prompt too long 的风险更高。 |
| `warningBuffer` | warning 提前量 | tokens | `20_000` | 在真正压缩前给 UI 和用户留足操作空间。 | prompt 超过 `warningThreshold` 时发 warning。 | 更早提醒，可能更打扰。 | 更晚提醒，用户更难处理。 |
| `maxCompactFailures` | 自动压缩连续失败熔断次数 | 次 | `3` | 3 次能覆盖偶发失败，同时避免无限循环和成本失控。 | 第 3 次失败后 block。 | 恢复机会更多，但成本更高。 | 更快失败，但偶发问题更容易中断任务。 |

## 公式解释

| 公式 | 变量解释 | 结果含义 | 后续动作 |
|---|---|---|---|
| `reserve = min(modelMaxOutputTokens, 20_000)` | `modelMaxOutputTokens` 是本次输出上限；`20_000` 是大模型预留封顶。 | 模型输出和压缩摘要最多预留多少空间。 | 用来计算有效窗口。 |
| `effectiveWindow = contextWindow - reserve` | `contextWindow` 是目标模型总窗口；`reserve` 是输出预留。 | prompt 真正能安全使用的空间。 | 所有阈值从这里派生。 |
| `autoCompactAt = effectiveWindow - 13_000` | `13_000` 是自动压缩缓冲。 | prompt 超过该值后，应先压缩再调用模型。 | 返回 `auto_compact`。 |
| `blockAt = effectiveWindow - 3_000` | `3_000` 是最后阻塞余量。 | prompt 超过该值后，直接调用模型风险太高。 | 返回 `full_compact` 或 `block`。 |
| `warningThreshold = autoCompactAt - 20_000` | `20_000` 是提前提醒空间。 | prompt 接近自动压缩线。 | 返回 `warn`，但不改写上下文。 |

## 详细流程

1. 估算当前 promptTokens。
2. 读取目标模型的 `contextWindow`，注意使用目标模型，不使用历史模型。
3. 如果目标模型窗口小于等于 `20_000`，reserve 使用 `min(modelMaxOutputTokens, 4_000)`。
4. 否则 reserve 使用 `min(modelMaxOutputTokens, 20_000)`。
5. 计算 `effectiveWindow`、`warningThreshold`、`autoCompactAt`、`blockAt`。
6. 如果 compactFailures 已达到 3，返回 `block`。
7. 如果正在从大模型切到小模型，并且 prompt 超过目标模型 `warningThreshold`，返回 `downgrade_compact`。
8. 如果 prompt 小于等于 warningThreshold，返回 `continue`。
9. 如果 prompt 小于等于 autoCompactAt，返回 `warn`。
10. 如果 prompt 小于等于 blockAt，返回 `auto_compact`。
11. 如果 prompt 大于 blockAt，返回 `full_compact`，禁止直接调用模型。

## 计算例子

### 200k 模型

| 步骤 | 计算 | 结果 | 解释 |
|---|---|---:|---|
| 总窗口 | `contextWindow` | `200_000` | 目标模型窗口。 |
| 输出预留 | `min(modelMaxOutputTokens, 20_000)` | `20_000` | 给模型输出和摘要留空间。 |
| 有效窗口 | `200_000 - 20_000` | `180_000` | prompt 安全可用空间。 |
| 自动压缩 | `180_000 - 13_000` | `167_000` | 超过后自动 compact。 |
| 阻塞限制 | `180_000 - 3_000` | `177_000` | 超过后禁止直接调用模型。 |
| warning | `167_000 - 20_000` | `147_000` | 超过后发 warning。 |

### 1M 模型

| 步骤 | 计算 | 结果 | 解释 |
|---|---|---:|---|
| 总窗口 | `contextWindow` | `1_000_000` | 大上下文窗口。 |
| 输出预留 | `min(modelMaxOutputTokens, 20_000)` | `20_000` | 大模型仍只预留 20k。 |
| 有效窗口 | `1_000_000 - 20_000` | `980_000` | prompt 安全可用空间。 |
| 自动压缩 | `980_000 - 13_000` | `967_000` | 超过后自动 compact。 |
| 阻塞限制 | `980_000 - 3_000` | `977_000` | 超过后 full compact。 |
| warning | `967_000 - 20_000` | `947_000` | 超过后发 warning。 |

### 20k 模型

| 步骤 | 计算 | 结果 | 解释 |
|---|---|---:|---|
| 总窗口 | `contextWindow` | `20_000` | 目标小模型窗口。 |
| 输出预留 | `min(modelMaxOutputTokens, 4_000)` | `4_000` | 小模型不能预留 20k。 |
| 有效窗口 | `20_000 - 4_000` | `16_000` | prompt 安全可用空间。 |
| 自动压缩 | `16_000 - 13_000` | `3_000` | 超过 3k 就应降级压缩。 |
| 阻塞限制 | `16_000 - 3_000` | `13_000` | 超过后不能直接调用。 |

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| `token_estimator_unavailable` | 用字符近似，并写 `estimate=true`；中文按每 1.5 字约 1 token，英文按每 4 字符约 1 token。 |
| `compact_failed_once` | 降低输出预算后重试一次，并记录 compactFailureCount。 |
| `compact_failed_three_times` | 熔断并 block，要求用户确认清理历史或换大模型。 |
| `provider_prompt_too_long` | reactive compact 只重试一次；再次失败就 block。 |
| `downgrade_still_too_large` | 第 1 次更激进 compact；第 2 次只保留 task state + latest user intent；第 3 次 block。 |

## 提示词模板

本节区分两类内容：

- `来源：Claude Code 源码内置`：来自 `/Users/sky/Downloads/claude-code-main` 的真实源码 prompt，保留英文原文，并在下面给出中文版本。
- `来源：本框架建议模板`：Claude Code 源码里没有同名自然语言 prompt，本框架为了复用实现而补充的中文模板或流程说明。

### Claude Code 源码内置提示词总览

| 名称 | 来源 | 调用路径 | 触发场景 |
|---|---|---|---|
| `NO_TOOLS_PREAMBLE` | `src/services/compact/prompt.ts` | `getCompactPrompt()`、`getPartialCompactPrompt()` | compact 模型开始摘要前，强制只输出文本。 |
| `DETAILED_ANALYSIS_INSTRUCTION_BASE` | `src/services/compact/prompt.ts` | `BASE_COMPACT_PROMPT`、`PARTIAL_COMPACT_UP_TO_PROMPT` | 摘要完整会话或摘要某个边界之前的历史。 |
| `DETAILED_ANALYSIS_INSTRUCTION_PARTIAL` | `src/services/compact/prompt.ts` | `PARTIAL_COMPACT_PROMPT` | 只摘要最近消息。 |
| `BASE_COMPACT_PROMPT` | `src/services/compact/prompt.ts` | `getCompactPrompt()` -> `compactConversation()` -> `/compact` | 传统完整 compact。 |
| `PARTIAL_COMPACT_PROMPT` | `src/services/compact/prompt.ts` | `getPartialCompactPrompt(customInstructions, "from")` | 保留早期上下文，只摘要最近部分。 |
| `PARTIAL_COMPACT_UP_TO_PROMPT` | `src/services/compact/prompt.ts` | `getPartialCompactPrompt(customInstructions, "up_to")` | 摘要某个边界之前内容，新消息随后继续。 |
| `NO_TOOLS_TRAILER` | `src/services/compact/prompt.ts` | compact prompt 末尾追加 | 再次提醒不要调用工具。 |
| `getCompactUserSummaryMessage()` | `src/services/compact/prompt.ts` | compact 成功后包装 summary message | 把摘要作为继续会话的 user message 放回上下文。 |

### `NO_TOOLS_PREAMBLE`

来源：Claude Code 源码内置，`src/services/compact/prompt.ts`。

触发场景：`getCompactPrompt()` 和 `getPartialCompactPrompt()` 生成压缩请求时放在最前面。

输出要求：只能输出纯文本，且必须包含 `<analysis>` 和 `<summary>`。

后处理逻辑：`formatCompactSummary()` 会删除 `<analysis>`，只把 `<summary>` 转成继续会话使用的摘要。

源码原文：

~~~text
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
~~~

中文版本：

~~~text
关键要求：只能用文本回复。不要调用任何工具。

- 不要使用 Read、Bash、Grep、Glob、Edit、Write 或任何其它工具。
- 你已经拥有上面对话中的全部必要上下文。
- 工具调用会被拒绝，并且会浪费你唯一的一轮机会，导致任务失败。
- 你的完整回复必须是纯文本：先输出一个 <analysis> 块，再输出一个 <summary> 块。
~~~

### `DETAILED_ANALYSIS_INSTRUCTION_BASE`

来源：Claude Code 源码内置，`src/services/compact/prompt.ts`。

触发场景：完整 compact，或 `up_to` 方向 partial compact。

输出要求：先在 `<analysis>` 中按时间顺序分析完整会话，再在 `<summary>` 中输出正式摘要。

后处理逻辑：`<analysis>` 是草稿区，`formatCompactSummary()` 会移除，不进入后续上下文。

源码原文：

~~~text
Before providing your final summary, wrap your analysis in <analysis> tags to organize your thoughts and ensure you've covered all necessary points. In your analysis process:

1. Chronologically analyze each message and section of the conversation. For each section thoroughly identify:
   - The user's explicit requests and intents
   - Your approach to addressing the user's requests
   - Key decisions, technical concepts and code patterns
   - Specific details like:
     - file names
     - full code snippets
     - function signatures
     - file edits
   - Errors that you ran into and how you fixed them
   - Pay special attention to specific user feedback that you received, especially if the user told you to do something differently.
2. Double-check for technical accuracy and completeness, addressing each required element thoroughly.
~~~

中文版本：

~~~text
在提供最终摘要之前，请先用 <analysis> 标签包裹你的分析过程，用来整理思路并确认所有必要点都已覆盖。分析时：

1. 按时间顺序分析对话中的每条消息和每个部分。对每个部分都要仔细识别：
   - 用户明确提出的请求和意图；
   - 你处理这些请求的方法；
   - 关键决策、技术概念和代码模式；
   - 具体细节，例如：
     - 文件名；
     - 完整代码片段；
     - 函数签名；
     - 文件编辑；
   - 你遇到的错误以及如何修复；
   - 特别关注用户给出的具体反馈，尤其是用户要求你换一种做法的地方。
2. 复查技术准确性和完整性，确保每个必要元素都被充分处理。
~~~

### `DETAILED_ANALYSIS_INSTRUCTION_PARTIAL`

来源：Claude Code 源码内置，`src/services/compact/prompt.ts`。

触发场景：`getPartialCompactPrompt(customInstructions, "from")`，只压缩最近消息时使用。

输出要求：只分析 recent messages，不要重复摘要已经保留的早期上下文。

后处理逻辑：同样由 `formatCompactSummary()` 删除 `<analysis>`，保留 `<summary>`。

源码原文：

~~~text
Before providing your final summary, wrap your analysis in <analysis> tags to organize your thoughts and ensure you've covered all necessary points. In your analysis process:

1. Analyze the recent messages chronologically. For each section thoroughly identify:
   - The user's explicit requests and intents
   - Your approach to addressing the user's requests
   - Key decisions, technical concepts and code patterns
   - Specific details like:
     - file names
     - full code snippets
     - function signatures
     - file edits
   - Errors that you ran into and how you fixed them
   - Pay special attention to specific user feedback that you received, especially if the user told you to do something differently.
2. Double-check for technical accuracy and completeness, addressing each required element thoroughly.
~~~

中文版本：

~~~text
在提供最终摘要之前，请先用 <analysis> 标签包裹你的分析过程，用来整理思路并确认所有必要点都已覆盖。分析时：

1. 按时间顺序分析最近消息。对每个部分都要仔细识别：
   - 用户明确提出的请求和意图；
   - 你处理这些请求的方法；
   - 关键决策、技术概念和代码模式；
   - 具体细节，例如文件名、完整代码片段、函数签名、文件编辑；
   - 你遇到的错误以及如何修复；
   - 特别关注用户给出的具体反馈，尤其是用户要求你换一种做法的地方。
2. 复查技术准确性和完整性，确保每个必要元素都被充分处理。
~~~

### `BASE_COMPACT_PROMPT`（full compact prompt / 完整压缩提示词）

来源：Claude Code 源码内置，`src/services/compact/prompt.ts`。

触发场景：传统完整 compact。`/compact` 命令或自动 compact 进入 `compactConversation()` 后，通过 `getCompactPrompt(customInstructions)` 生成。

输出要求：生成 `<analysis>` 和 `<summary>`，summary 必须包含 9 个固定章节。

后处理逻辑：`formatCompactSummary()` 移除 `<analysis>`；`getCompactUserSummaryMessage()` 将 summary 包成继续会话消息。

源码原文：下面是该源码常量的固定正文。运行时 `getCompactPrompt()` 会在前面拼接 `NO_TOOLS_PREAMBLE`，在后面按需拼接 `Additional Instructions` 和 `NO_TOOLS_TRAILER`。

~~~text
Your task is to create a detailed summary of the conversation so far, paying close attention to the user's explicit requests and your previous actions.
This summary should be thorough in capturing technical details, code patterns, and architectural decisions that would be essential for continuing development work without losing context.

Your summary should include the following sections:

1. Primary Request and Intent: Capture all of the user's explicit requests and intents in detail
2. Key Technical Concepts: List all important technical concepts, technologies, and frameworks discussed.
3. Files and Code Sections: Enumerate specific files and code sections examined, modified, or created. Pay special attention to the most recent messages and include full code snippets where applicable and include a summary of why this file read or edit is important.
4. Errors and fixes: List all errors that you ran into, and how you fixed them. Pay special attention to specific user feedback that you received, especially if the user told you to do something differently.
5. Problem Solving: Document problems solved and any ongoing troubleshooting efforts.
6. All user messages: List ALL user messages that are not tool results. These are critical for understanding the users' feedback and changing intent.
7. Pending Tasks: Outline any pending tasks that you have explicitly been asked to work on.
8. Current Work: Describe in detail precisely what was being worked on immediately before this summary request, paying special attention to the most recent messages from both user and assistant. Include file names and code snippets where applicable.
9. Optional Next Step: List the next step that you will take that is related to the most recent work you were doing. IMPORTANT: ensure that this step is DIRECTLY in line with the user's most recent explicit requests, and the task you were working on immediately before this summary request. If your last task was concluded, then only list next steps if they are explicitly in line with the users request. Do not start on tangential requests or really old requests that were already completed without confirming with the user first.
                       If there is a next step, include direct quotes from the most recent conversation showing exactly what task you were working on and where you left off. This should be verbatim to ensure there's no drift in task interpretation.

Here's an example of how your output should be structured:

<example>
<analysis>
[Your thought process, ensuring all points are covered thoroughly and accurately]
</analysis>

<summary>
1. Primary Request and Intent:
   [Detailed description]

2. Key Technical Concepts:
   - [Concept 1]
   - [Concept 2]
   - [...]

3. Files and Code Sections:
   - [File Name 1]
      - [Summary of why this file is important]
      - [Summary of the changes made to this file, if any]
      - [Important Code Snippet]
   - [File Name 2]
      - [Important Code Snippet]
   - [...]

4. Errors and fixes:
    - [Detailed description of error 1]:
      - [How you fixed the error]
      - [User feedback on the error if any]
    - [...]

5. Problem Solving:
   [Description of solved problems and ongoing troubleshooting]

6. All user messages: 
    - [Detailed non tool use user message]
    - [...]

7. Pending Tasks:
   - [Task 1]
   - [Task 2]
   - [...]

8. Current Work:
   [Precise description of current work]

9. Optional Next Step:
   [Optional Next step to take]

</summary>
</example>

Please provide your summary based on the conversation so far, following this structure and ensuring precision and thoroughness in your response.

There may be additional summarization instructions provided in the included context. If so, remember to follow these instructions when creating the above summary. Examples of instructions include:
<example>
## Compact Instructions
When summarizing the conversation focus on typescript code changes and also remember the mistakes you made and how you fixed them.
</example>

<example>
# Summary instructions
When you are using compact - please focus on test output and code changes. Include file reads verbatim.
</example>
~~~

中文版本：

~~~text
你的任务是为目前为止的对话创建一份详细摘要，特别关注用户明确提出的请求以及你之前采取的行动。
这份摘要必须充分捕捉技术细节、代码模式和架构决策，因为这些信息对于后续继续开发而不丢失上下文非常重要。

摘要必须包含以下章节：

1. 主要请求与意图：详细记录用户所有明确请求和意图。
2. 关键技术概念：列出讨论过的重要技术概念、技术栈和框架。
3. 文件与代码片段：列出查看、修改或创建过的具体文件和代码区域。特别关注最近消息；适用时包含完整代码片段，并说明为什么该文件读取或编辑重要。
4. 错误与修复：列出遇到的所有错误以及修复方式。特别关注用户反馈，尤其是用户要求改变做法的地方。
5. 问题解决：记录已经解决的问题和仍在排查的问题。
6. 所有用户消息：列出所有不是工具结果的用户消息。这些消息对理解用户反馈和意图变化至关重要。
7. 待办任务：列出用户明确要求处理但尚未完成的任务。
8. 当前工作：详细说明摘要请求前正在做什么，特别关注最近的用户和助手消息。适用时包含文件名和代码片段。
9. 可选下一步：列出与最近工作直接相关的下一步。必须直接符合用户最近的明确请求和摘要前正在做的任务。如果上一个任务已经完成，只有当下一步明确符合用户请求时才列出。不要在未确认的情况下开始无关任务或很久以前已经完成的旧任务。

输出结构示例：

<example>
<analysis>
[你的分析过程，确保所有点都被完整准确覆盖]
</analysis>

<summary>
1. 主要请求与意图：
   [详细描述]

2. 关键技术概念：
   - [概念 1]
   - [概念 2]
   - [...]

3. 文件与代码片段：
   - [文件名 1]
      - [为什么这个文件重要]
      - [如果修改过，说明修改内容]
      - [重要代码片段]

4. 错误与修复：
   - [错误 1 的详细描述]：
      - [如何修复]
      - [用户是否对该错误给过反馈]

5. 问题解决：
   [已解决问题和仍在排查的问题]

6. 所有用户消息：
   - [不是工具结果的用户消息]

7. 待办任务：
   - [任务 1]

8. 当前工作：
   [当前工作的精确描述]

9. 可选下一步：
   [可选下一步]
</summary>
</example>

请根据目前为止的对话，按照上述结构提供摘要，并确保准确、完整、细致。

如果上下文中包含额外摘要指令，也必须遵守。例如用户可能要求 compact 时重点关注 TypeScript 代码改动、测试输出、代码变更，或逐字包含文件读取内容。
~~~

### `PARTIAL_COMPACT_PROMPT`

来源：Claude Code 源码内置，`src/services/compact/prompt.ts`。

触发场景：保留早期上下文，只压缩后续 recent messages。对应 `getPartialCompactPrompt(customInstructions, "from")`。

输出要求：只总结 recent portion，不总结已保留的 earlier messages。

后处理逻辑：生成的 summary 和保留的 earlier messages 一起组成新上下文。

源码原文：下面是该源码常量的固定正文。运行时 `getPartialCompactPrompt(customInstructions, "from")` 会在前面拼接 `NO_TOOLS_PREAMBLE`，在后面按需拼接 `Additional Instructions` 和 `NO_TOOLS_TRAILER`。

~~~text
Your task is to create a detailed summary of the RECENT portion of the conversation — the messages that follow earlier retained context. The earlier messages are being kept intact and do NOT need to be summarized. Focus your summary on what was discussed, learned, and accomplished in the recent messages only.

Your summary should include the following sections:

1. Primary Request and Intent: Capture the user's explicit requests and intents from the recent messages
2. Key Technical Concepts: List important technical concepts, technologies, and frameworks discussed recently.
3. Files and Code Sections: Enumerate specific files and code sections examined, modified, or created. Include full code snippets where applicable and include a summary of why this file read or edit is important.
4. Errors and fixes: List errors encountered and how they were fixed.
5. Problem Solving: Document problems solved and any ongoing troubleshooting efforts.
6. All user messages: List ALL user messages from the recent portion that are not tool results.
7. Pending Tasks: Outline any pending tasks from the recent messages.
8. Current Work: Describe precisely what was being worked on immediately before this summary request.
9. Optional Next Step: List the next step related to the most recent work. Include direct quotes from the most recent conversation.

Here's an example of how your output should be structured:

<example>
<analysis>
[Your thought process, ensuring all points are covered thoroughly and accurately]
</analysis>

<summary>
1. Primary Request and Intent:
   [Detailed description]

2. Key Technical Concepts:
   - [Concept 1]
   - [Concept 2]

3. Files and Code Sections:
   - [File Name 1]
      - [Summary of why this file is important]
      - [Important Code Snippet]

4. Errors and fixes:
    - [Error description]:
      - [How you fixed it]

5. Problem Solving:
   [Description]

6. All user messages:
    - [Detailed non tool use user message]

7. Pending Tasks:
   - [Task 1]

8. Current Work:
   [Precise description of current work]

9. Optional Next Step:
   [Optional Next step to take]

</summary>
</example>

Please provide your summary based on the RECENT messages only (after the retained earlier context), following this structure and ensuring precision and thoroughness in your response.
~~~

中文版本：

~~~text
你的任务是为对话中最近的一段创建详细摘要，也就是早期保留上下文之后的消息。早期消息会被原样保留，不需要摘要。你的摘要只关注最近消息里讨论、学到和完成的内容。

摘要必须包含以下章节：

1. 主要请求与意图：记录最近消息中用户明确提出的请求和意图。
2. 关键技术概念：列出最近讨论的重要技术概念、技术栈和框架。
3. 文件与代码片段：列出最近查看、修改或创建过的具体文件和代码区域。适用时包含完整代码片段，并说明为什么该文件读取或编辑重要。
4. 错误与修复：列出最近遇到的错误以及修复方式。
5. 问题解决：记录最近解决的问题和仍在排查的问题。
6. 所有用户消息：列出最近部分中所有不是工具结果的用户消息。
7. 待办任务：列出最近消息中的待办任务。
8. 当前工作：准确说明摘要请求前正在做什么。
9. 可选下一步：列出与最近工作相关的下一步，并包含最近对话中的直接引用。

输出结构示例：

<example>
<analysis>
[你的分析过程，确保所有点都被完整准确覆盖]
</analysis>

<summary>
1. 主要请求与意图：
   [详细描述]

2. 关键技术概念：
   - [概念 1]
   - [概念 2]

3. 文件与代码片段：
   - [文件名 1]
      - [为什么这个文件重要]
      - [重要代码片段]

4. 错误与修复：
   - [错误描述]：
      - [如何修复]

5. 问题解决：
   [描述]

6. 所有用户消息：
   - [不是工具结果的用户消息]

7. 待办任务：
   - [任务 1]

8. 当前工作：
   [当前工作的精确描述]

9. 可选下一步：
   [可选下一步]
</summary>
</example>

请只根据保留早期上下文之后的最近消息生成摘要，并确保准确、完整、细致。
~~~

### `PARTIAL_COMPACT_UP_TO_PROMPT`

来源：Claude Code 源码内置，`src/services/compact/prompt.ts`。

触发场景：摘要某个边界之前的内容，较新的消息会跟在 summary 后继续。对应 `getPartialCompactPrompt(customInstructions, "up_to")`。

输出要求：summary 必须足够完整，让只读 summary 和后续较新消息的人也能继续工作。

后处理逻辑：summary 放在 continuing session 开头，后续新消息原样跟在后面。

源码原文：下面是该源码常量的固定正文。运行时 `getPartialCompactPrompt(customInstructions, "up_to")` 会在前面拼接 `NO_TOOLS_PREAMBLE`，在后面按需拼接 `Additional Instructions` 和 `NO_TOOLS_TRAILER`。

~~~text
Your task is to create a detailed summary of this conversation. This summary will be placed at the start of a continuing session; newer messages that build on this context will follow after your summary (you do not see them here). Summarize thoroughly so that someone reading only your summary and then the newer messages can fully understand what happened and continue the work.

Your summary should include the following sections:

1. Primary Request and Intent: Capture the user's explicit requests and intents in detail
2. Key Technical Concepts: List important technical concepts, technologies, and frameworks discussed.
3. Files and Code Sections: Enumerate specific files and code sections examined, modified, or created. Include full code snippets where applicable and include a summary of why this file read or edit is important.
4. Errors and fixes: List errors encountered and how they were fixed.
5. Problem Solving: Document problems solved and any ongoing troubleshooting efforts.
6. All user messages: List ALL user messages that are not tool results.
7. Pending Tasks: Outline any pending tasks.
8. Work Completed: Describe what was accomplished by the end of this portion.
9. Context for Continuing Work: Summarize any context, decisions, or state that would be needed to understand and continue the work in subsequent messages.

Here's an example of how your output should be structured:

<example>
<analysis>
[Your thought process, ensuring all points are covered thoroughly and accurately]
</analysis>

<summary>
1. Primary Request and Intent:
   [Detailed description]

2. Key Technical Concepts:
   - [Concept 1]
   - [Concept 2]

3. Files and Code Sections:
   - [File Name 1]
      - [Summary of why this file is important]
      - [Important Code Snippet]

4. Errors and fixes:
    - [Error description]:
      - [How you fixed it]

5. Problem Solving:
   [Description]

6. All user messages:
    - [Detailed non tool use user message]

7. Pending Tasks:
   - [Task 1]

8. Work Completed:
   [Description of what was accomplished]

9. Context for Continuing Work:
   [Key context, decisions, or state needed to continue the work]

</summary>
</example>

Please provide your summary following this structure, ensuring precision and thoroughness in your response.
~~~

中文版本：

~~~text
你的任务是为这段对话创建详细摘要。这个摘要会被放在继续会话的开头；基于这段上下文产生的更新消息会跟在摘要后面，但你现在看不到那些新消息。请充分摘要，让只阅读你的摘要和后续新消息的人也能完全理解之前发生了什么，并继续工作。

摘要必须包含以下章节：

1. 主要请求与意图：详细记录用户明确提出的请求和意图。
2. 关键技术概念：列出讨论过的重要技术概念、技术栈和框架。
3. 文件与代码片段：列出查看、修改或创建过的具体文件和代码区域。适用时包含完整代码片段，并说明为什么该文件读取或编辑重要。
4. 错误与修复：列出遇到的错误以及修复方式。
5. 问题解决：记录已经解决的问题和仍在排查的问题。
6. 所有用户消息：列出所有不是工具结果的用户消息。
7. 待办任务：列出待办任务。
8. 已完成工作：说明到这部分结束时完成了什么。
9. 继续工作所需上下文：总结后续消息需要依赖的上下文、决策或状态。

输出结构示例：

<example>
<analysis>
[你的分析过程，确保所有点都被完整准确覆盖]
</analysis>

<summary>
1. 主要请求与意图：
   [详细描述]

2. 关键技术概念：
   - [概念 1]
   - [概念 2]

3. 文件与代码片段：
   - [文件名 1]
      - [为什么这个文件重要]
      - [重要代码片段]

4. 错误与修复：
   - [错误描述]：
      - [如何修复]

5. 问题解决：
   [描述]

6. 所有用户消息：
   - [不是工具结果的用户消息]

7. 待办任务：
   - [任务 1]

8. 已完成工作：
   [完成内容]

9. 继续工作所需上下文：
   [继续工作所需的关键上下文、决策或状态]
</summary>
</example>

请按照上述结构提供摘要，并确保准确、完整、细致。
~~~

### `NO_TOOLS_TRAILER`

来源：Claude Code 源码内置，`src/services/compact/prompt.ts`。

触发场景：`getCompactPrompt()` 和 `getPartialCompactPrompt()` 在 prompt 末尾追加。

输出要求：再次约束只输出文本，不调用工具。

源码原文：

~~~text
REMINDER: Do NOT call any tools. Respond with plain text only — an <analysis> block followed by a <summary> block. Tool calls will be rejected and you will fail the task.
~~~

中文版本：

~~~text
提醒：不要调用任何工具。只能用纯文本回复，先输出 <analysis> 块，再输出 <summary> 块。工具调用会被拒绝，并导致任务失败。
~~~

### `getCompactUserSummaryMessage()`

来源：Claude Code 源码内置，`src/services/compact/prompt.ts`。

触发场景：compact 生成 summary 后，把 summary 包装成继续会话的 user message。

输出要求：告诉后续模型“这是从上一个因为上下文耗尽而继续的会话”，并可附加 transcript 路径、recent messages 是否保留、是否禁止追问。

后处理逻辑：如果 `suppressFollowUpQuestions=true`，会追加“直接继续，不要问用户、不用复述”的指令；proactive 模式还会追加自主继续工作循环的说明。

源码原文：

~~~text
This session is being continued from a previous conversation that ran out of context. The summary below covers the earlier portion of the conversation.

{formattedSummary}

If you need specific details from before compaction (like exact code snippets, error messages, or content you generated), read the full transcript at: {transcriptPath}

Recent messages are preserved verbatim.

Continue the conversation from where it left off without asking the user any further questions. Resume directly — do not acknowledge the summary, do not recap what was happening, do not preface with "I'll continue" or similar. Pick up the last task as if the break never happened.
~~~

中文版本：

~~~text
本会话是从之前一个已经耗尽上下文的对话继续而来。下面的摘要覆盖了较早的对话部分。

{格式化后的摘要}

如果你需要压缩前的具体细节，例如精确代码片段、错误消息或你生成过的内容，请读取完整 transcript：{transcriptPath}

最近消息已逐字保留。

从会话中断处直接继续，不要再向用户追问。直接恢复，不要确认摘要，不要复述之前发生了什么，也不要说“我会继续”之类的开场白。像上下文中断从未发生一样，接着最后的任务继续做。
~~~

### Session Memory 相关提示词

来源：Claude Code 源码内置，`src/services/SessionMemory/prompts.ts`。

触发场景：Session Memory 功能启用时，用 forked agent 更新会话记忆；`/summary` 也会手动触发 session memory extraction。

关键数字：

| 名称 | 默认值 | 含义 |
|---|---:|---|
| `MAX_SECTION_LENGTH` | `2000` | 单个 session memory section 的软上限。 |
| `MAX_TOTAL_SESSION_MEMORY_TOKENS` | `12000` | session memory 文件整体 token 上限。 |

#### `DEFAULT_SESSION_MEMORY_TEMPLATE`

源码原文：

~~~text
# Session Title
_A short and distinctive 5-10 word descriptive title for the session. Super info dense, no filler_

# Current State
_What is actively being worked on right now? Pending tasks not yet completed. Immediate next steps._

# Task specification
_What did the user ask to build? Any design decisions or other explanatory context_

# Files and Functions
_What are the important files? In short, what do they contain and why are they relevant?_

# Workflow
_What bash commands are usually run and in what order? How to interpret their output if not obvious?_

# Errors & Corrections
_Errors encountered and how they were fixed. What did the user correct? What approaches failed and should not be tried again?_

# Codebase and System Documentation
_What are the important system components? How do they work/fit together?_

# Learnings
_What has worked well? What has not? What to avoid? Do not duplicate items from other sections_

# Key results
_If the user asked a specific output such as an answer to a question, a table, or other document, repeat the exact result here_

# Worklog
_Step by step, what was attempted, done? Very terse summary for each step_
~~~

中文版本：

~~~text
# 会话标题
_用 5-10 个词写一个简短、有辨识度、信息密度高、没有废话的会话标题_

# 当前状态
_现在正在做什么？还有哪些未完成任务？下一步马上要做什么？_

# 任务规格
_用户要求构建什么？有哪些设计决策或解释性上下文？_

# 文件与函数
_重要文件有哪些？简要说明它们包含什么以及为什么相关。_

# 工作流
_通常运行哪些 bash 命令，顺序是什么？如果输出不明显，应如何理解？_

# 错误与纠正
_遇到过哪些错误以及如何修复？用户纠正了什么？哪些失败方案不应再尝试？_

# 代码库与系统文档
_重要系统组件有哪些？它们如何工作、如何组合？_

# 经验教训
_哪些做法有效？哪些无效？应避免什么？不要和其它章节重复。_

# 关键结果
_如果用户要求了具体输出，例如答案、表格或文档，请在这里重复精确结果。_

# 工作日志
_按步骤极简记录尝试过什么、完成了什么。_
~~~

#### `getDefaultUpdatePrompt()`

源码原文：

~~~text
IMPORTANT: This message and these instructions are NOT part of the actual user conversation. Do NOT include any references to "note-taking", "session notes extraction", or these update instructions in the notes content.

Based on the user conversation above (EXCLUDING this note-taking instruction message as well as system prompt, claude.md entries, or any past session summaries), update the session notes file.

The file {{notesPath}} has already been read for you. Here are its current contents:
<current_notes_content>
{{currentNotes}}
</current_notes_content>

Your ONLY task is to use the Edit tool to update the notes file, then stop. You can make multiple edits (update every section as needed) - make all Edit tool calls in parallel in a single message. Do not call any other tools.

CRITICAL RULES FOR EDITING:
- The file must maintain its exact structure with all sections, headers, and italic descriptions intact
-- NEVER modify, delete, or add section headers (the lines starting with '#' like # Task specification)
-- NEVER modify or delete the italic _section description_ lines (these are the lines in italics immediately following each header - they start and end with underscores)
-- The italic _section descriptions_ are TEMPLATE INSTRUCTIONS that must be preserved exactly as-is - they guide what content belongs in each section
-- ONLY update the actual content that appears BELOW the italic _section descriptions_ within each existing section
-- Do NOT add any new sections, summaries, or information outside the existing structure
- Do NOT reference this note-taking process or instructions anywhere in the notes
- It's OK to skip updating a section if there are no substantial new insights to add. Do not add filler content like "No info yet", just leave sections blank/unedited if appropriate.
- Write DETAILED, INFO-DENSE content for each section - include specifics like file paths, function names, error messages, exact commands, technical details, etc.
- For "Key results", include the complete, exact output the user requested (e.g., full table, full answer, etc.)
- Do not include information that's already in the CLAUDE.md files included in the context
- Keep each section under ~2000 tokens/words - if a section is approaching this limit, condense it by cycling out less important details while preserving the most critical information
- Focus on actionable, specific information that would help someone understand or recreate the work discussed in the conversation
- IMPORTANT: Always update "Current State" to reflect the most recent work - this is critical for continuity after compaction

Use the Edit tool with file_path: {{notesPath}}

STRUCTURE PRESERVATION REMINDER:
Each section has TWO parts that must be preserved exactly as they appear in the current file:
1. The section header (line starting with #)
2. The italic description line (the _italicized text_ immediately after the header - this is a template instruction)

You ONLY update the actual content that comes AFTER these two preserved lines. The italic description lines starting and ending with underscores are part of the template structure, NOT content to be edited or removed.

REMEMBER: Use the Edit tool in parallel and stop. Do not continue after the edits. Only include insights from the actual user conversation, never from these note-taking instructions. Do not delete or change section headers or italic _section descriptions_.
~~~

中文版本：

~~~text
重要：这条消息和这些指令不是实际用户对话的一部分。不要在笔记内容中提到“记笔记”“会话笔记抽取”或这些更新指令。

基于上方用户对话更新会话笔记文件，但要排除这条记笔记指令、系统提示、CLAUDE.md 条目以及任何过去的会话摘要。

文件 {{notesPath}} 已经为你读取。当前内容如下：
<current_notes_content>
{{currentNotes}}
</current_notes_content>

你的唯一任务是使用 Edit 工具更新笔记文件，然后停止。你可以进行多处编辑，按需要更新每个章节；所有 Edit 工具调用应在同一条消息中并行发出。不要调用其它工具。

关键编辑规则：
- 文件必须保持完全相同的结构，所有章节、标题和斜体说明都要保留。
-- 永远不要修改、删除或新增章节标题。
-- 永远不要修改或删除每个标题后面的斜体章节说明。
-- 斜体说明是模板指令，必须原样保留，用来指导每个章节应该写什么。
-- 只能更新每个已有章节中斜体说明下面的实际内容。
-- 不要在现有结构之外新增章节、摘要或信息。
- 不要在笔记中提到记笔记过程或这些指令。
- 如果某个章节没有重要新信息，可以跳过，不要写“暂无信息”之类的填充内容。
- 每个章节都要写得详细、信息密度高，包括文件路径、函数名、错误消息、精确命令、技术细节等。
- 对于“Key results”，要包含用户要求的完整精确输出。
- 不要包含已经在上下文里的 CLAUDE.md 文件中存在的信息。
- 每个章节保持在约 2000 tokens/words 以下；接近上限时，删除较不重要的细节，保留最关键的信息。
- 聚焦可执行、具体的信息，让后续读者能理解或复现对话中的工作。
- 重要：必须始终更新“Current State”，反映最近工作；这对压缩后的连续性非常关键。

使用 Edit 工具，file_path 为 {{notesPath}}。

结构保留提醒：
每个章节都有两个必须原样保留的部分：
1. 章节标题。
2. 标题后面的斜体说明。

你只能更新这两个保留部分之后的实际内容。

记住：并行使用 Edit 工具后停止。不要在编辑后继续。只包含真实用户对话中的信息，永远不要包含这些记笔记指令。不要删除或修改标题和斜体说明。
~~~

#### 超预算提醒

源码原文：

~~~text
CRITICAL: The session memory file is currently ~{totalTokens} tokens, which exceeds the maximum of 12000 tokens. You MUST condense the file to fit within this budget. Aggressively shorten oversized sections by removing less important details, merging related items, and summarizing older entries. Prioritize keeping "Current State" and "Errors & Corrections" accurate and detailed.
~~~

中文版本：

~~~text
关键要求：当前 session memory 文件约为 {totalTokens} tokens，超过 12000 tokens 上限。你必须压缩该文件，使它符合预算。要积极缩短超大的章节，删除较不重要的细节、合并相关项目、摘要较旧记录。优先保持“Current State”和“Errors & Corrections”准确且详细。
~~~

### 不是 Claude Code 源码内置自然语言 prompt 的项目

#### 降级压缩提示词（downgrade compact prompt）

来源：本框架建议模板，不是 Claude Code 源码内置同名 prompt。

Claude Code 源码里没有名为 `downgrade compact prompt` 的自然语言提示词。相关行为主要由 partial compact、context management、模型窗口选择、`getEffectiveContextWindowSize()`、`getAutoCompactThreshold()` 和 prompt too long 恢复逻辑共同完成。

框架建议中文模板：

~~~text
你是降级压缩器。目标模型上下文更小，不能保留完整历史。
只保留：最新用户意图、当前任务状态、关键文件、未完成任务、影响下一步的工具结果摘要、风险和不能重复执行的副作用。
删除：完整聊天历史、旧计划、无关工具输出、可重新读取的长文件内容。
如果缺少信息，后续 Agent 应通过工具重新读取，不要靠猜。
~~~

#### 工具结果微压缩提示词（tool-result microcompact prompt）

来源：本框架建议模板，不是 Claude Code 源码内置同名 prompt。

Claude Code 的 microcompact 主要是代码策略或 API context management，不是普通自然语言 prompt。相关源码：

- `src/services/compact/microCompact.ts`
- `src/services/compact/apiMicrocompact.ts`

关键源码默认值：

| 名称 | 默认值 | 含义 |
|---|---:|---|
| `DEFAULT_MAX_INPUT_TOKENS` | `180_000` | API microcompact 触发输入 token 阈值。 |
| `DEFAULT_TARGET_INPUT_TOKENS` | `40_000` | API microcompact 后希望保留的输入目标。 |

框架建议中文模板：

~~~text
把工具输出压缩为后续决策需要的最小事实集。
输出：结论、关键证据、后续影响、原始输出位置。
删除：重复日志、安装进度、无关警告、长列表中的低价值项目。
~~~

#### 压缩失败恢复提示词（compact failure recovery prompt）

来源：本框架建议模板，不是 Claude Code 源码内置同名 prompt。

Claude Code 源码中 compact 失败恢复主要是流程控制：检测 no summary、incomplete response、prompt too long、abort、fallback streaming 等错误，并决定重试、截断头部或报错。它不是一个独立命名的自然语言 prompt。

框架建议中文模板：

~~~text
上一次压缩后仍超限。请执行更激进压缩。
只保留：最新用户意图、当前任务状态、下一步必须知道的 5 条以内事实、不能丢失的风险或权限限制。
输出必须少于上一次压缩结果的 50%。
~~~

#### 多 Agent 总结提示词（multi-agent summary prompt）

来源：本框架建议模板，不是 Claude Code compact 源码内置 prompt。

Claude Code 中与 Agent summary 相关的源码在 `src/services/AgentSummary/agentSummary.ts`，它是 coordinator/sub-agent 的周期性背景摘要，目标是生成短进度摘要，不等同于 conversation compact prompt。

框架建议中文模板：

~~~text
总结子 Agent 的任务、使用工具、读过或改过的文件、确定事实、失败、不确定项、风险和建议下一步。
禁止包含完整聊天历史。禁止把猜测写成事实。最多 8_000 tokens。
~~~

## 可实现伪代码

~~~ts
function decideContextAction(input: ContextBudgetInput): ContextBudgetDecision {
  const reserveCap = input.contextWindow <= 20_000 ? 4_000 : 20_000
  const reserve = Math.min(input.modelMaxOutputTokens, reserveCap)
  const effectiveWindow = input.contextWindow - reserve
  const autoCompactAt = effectiveWindow - 13_000
  const blockAt = effectiveWindow - 3_000
  const warningThreshold = autoCompactAt - 20_000

  if (input.compactFailures >= 3) {
    return { action: "block", promptTokens: input.promptTokens, reason: "compact_failed_three_times" }
  }
  if (input.isDowngrade && input.promptTokens > warningThreshold) {
    return { action: "downgrade_compact", promptTokens: input.promptTokens, targetWindow: input.contextWindow, targetModel: input.targetModel }
  }
  if (input.promptTokens <= warningThreshold) {
    return { action: "continue", promptTokens: input.promptTokens, warningThreshold, autoCompactAt, blockAt }
  }
  if (input.promptTokens <= autoCompactAt) {
    return { action: "warn", promptTokens: input.promptTokens, warningThreshold }
  }
  if (input.promptTokens <= blockAt) {
    return { action: "auto_compact", promptTokens: input.promptTokens, autoCompactAt }
  }
  return { action: "full_compact", promptTokens: input.promptTokens, blockAt }
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 200k 安全区 | `promptTokens=140_000` | `continue`。 |
| 200k warning | `promptTokens=150_000` | `warn`。 |
| 200k 自动压缩 | `promptTokens=170_000` | `auto_compact`。 |
| 200k 完整压缩 | `promptTokens=178_000` | `full_compact`，不调用模型。 |
| 1M 自动压缩 | `promptTokens=970_000` | `auto_compact`。 |
| 1M 完整压缩 | `promptTokens=978_000` | `full_compact`。 |
| 20k 降级 | 从 200k 会话切到 20k，`promptTokens=50_000` | `downgrade_compact`。 |
| compact 三次失败 | `compactFailures=3` | `block`。 |

## 验收标准

- 文档解释 `effectiveWindow`、`autoCompactAt`、`blockAt`、`warningThreshold` 每个变量的含义。
- 包含 `200_000`、`1_000_000`、`20_000`、`13_000`、`3_000`、`167_000`、`177_000`、`967_000`、`977_000`。
- 包含 200k、1M、20k 三个完整计算例子。
- 包含 downgrade compact、full compact prompt、tool-result microcompact prompt、multi-agent summary prompt、compact failure recovery prompt。
- blockAt 以上绝不调用模型。
