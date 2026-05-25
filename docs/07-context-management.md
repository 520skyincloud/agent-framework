# 上下文管理 / Context Management

## 本章目标 / Goals

说明 token 预算、自动压缩、手动阻塞、工具结果瘦身和反模式。

Explain token budgets, auto-compaction, blocking, tool result slimming, and anti-patterns.

完成本章后，读者应该知道这一层为什么存在、如何实现最小版本、哪些默认值不能随意改、以及如何验收。

After this chapter, the reader should know why this layer exists, how to implement the minimal version, which defaults must not be changed casually, and how to validate it.

## 核心概念 / Core Concepts

- 上下文管理必须提前行动。
- 本章能力必须有清晰的输入、输出、失败语义和测试边界。
- 任何影响模型下一步行为的状态，都必须能被记录、恢复或回放。

- Context management must act before provider rejection.
- This capability must have clear input, output, failure semantics, and test boundaries.
- Any state that affects the model's next action must be recordable, recoverable, or replayable.

## 架构位置 / Where This Fits

本章位于 `context` 层。它不是孤立模块，而是和主循环、工具系统、上下文管理、持久化、权限或 SDK 事件流共同工作。实现时要明确本层是否拥有状态，是否会产生副作用，是否会改变下一轮模型输入。

This chapter belongs to the `context` layer. It is not an isolated module; it works with the main loop, tool system, context management, persistence, permissions, or SDK event stream. Implementation must make clear whether this layer owns state, produces side effects, or changes the next model input.

## 具体设计 / Concrete Design

最小设计应包含四个部分：输入对象、输出对象、错误对象和持久化记录。输入对象用于阻止隐式全局依赖；输出对象用于让 UI、SDK 和 replay 共用同一结果；错误对象用于让模型或用户知道下一步怎么恢复；持久化记录用于崩溃后继续运行。

The minimal design should contain four parts: input object, output object, error object, and persistence record. The input object prevents hidden global dependencies; the output object lets UI, SDK, and replay share the same result; the error object tells the model or user how to recover; the persistence record lets execution continue after a crash.

成熟设计还应该补充可观测性和预算控制。只要本章能力可能变慢、变贵、失败或产生副作用，就必须发出事件并记录关键 ID。

A mature design should also include observability and budget control. If this capability can become slow, expensive, fail, or produce side effects, it must emit events and record key IDs.

## 接口与数据结构 / Interfaces And Data Structures

| 边界 / Boundary | 中文说明 | English Description |
|---|---|---|
| Input | 调用方必须显式传入的状态和参数。 | Explicit state and parameters required from the caller. |
| Output | 成功时返回的数据、事件或状态变更。 | Data, events, or state changes returned on success. |
| Error | 失败时返回给用户、模型或调用方的结构化错误。 | Structured errors returned to the user, model, or caller. |
| Persistence | 必须写入 transcript、metadata 或输出文件的内容。 | Content that must be written to transcript, metadata, or output files. |
| Replay | 回放测试需要记录和模拟的输入输出。 | Inputs and outputs replay tests must record and simulate. |

建议接口命名保持直接，例如 `contextConfig`、`contextState`、`contextEvent`、`contextResult`。如果这些类型变得过大，优先拆分所有权，而不是把所有字段塞进一个全局对象。

Use direct names such as `contextConfig`, `contextState`, `contextEvent`, and `contextResult`. If these types become too large, split ownership instead of putting every field into one global object.

## 默认值与关键数字 / Defaults And Numbers

| 配置 / Config | 默认值 / Default | 说明 / Notes |
|---|---:|---|
| `contextWindowTokens` | `200_000` | 实现时显式引用，不要隐藏在业务逻辑中。 / Reference explicitly in implementation; do not hide it in business logic. |
| `summaryReserveTokens` | `20_000` | 实现时显式引用，不要隐藏在业务逻辑中。 / Reference explicitly in implementation; do not hide it in business logic. |
| `autoCompactBufferTokens` | `13_000` | 实现时显式引用，不要隐藏在业务逻辑中。 / Reference explicitly in implementation; do not hide it in business logic. |
| `manualReserveTokens` | `3_000` | 实现时显式引用，不要隐藏在业务逻辑中。 / Reference explicitly in implementation; do not hide it in business logic. |

如果本章没有专属数字，就使用 `.agent/config.json` 中的全局默认值，并在实现中显式引用。不要让默认值只存在于文档里。

If this chapter has no dedicated numbers, use the global defaults in `.agent/config.json` and reference them explicitly in implementation. Defaults should not live only in documentation.

## 实现步骤 / Implementation Steps

1. 先实现最小闭环，再添加高级能力。
2. 定义输入、输出、错误和持久化边界。
3. 把默认值集中到配置或常量模块。
4. 为正常路径、失败路径和边界值写测试。
5. 把影响下一轮模型行为的状态写入 transcript、metadata 或 replay fixture。

1. Implement the minimal closed loop before adding advanced capabilities.
2. Define input, output, error, and persistence boundaries.
3. Centralize defaults in config or constants.
4. Write tests for happy paths, failure paths, and boundary values.
5. Write state that affects the next model turn into transcript, metadata, or replay fixtures.

## 测试与验收 / Tests And Acceptance Criteria

- 正常路径必须产出符合接口的结果。
- 失败路径必须返回结构化错误，而不是静默失败。
- 达到默认限制时必须触发文档规定的行为。
- 恢复或回放时结果必须可解释。
- 相关验收标准必须能被自动化测试验证。

- The happy path must produce results matching the interface.
- Failure paths must return structured errors instead of failing silently.
- At default limits, behavior must match this document.
- Resume or replay results must be explainable.
- Acceptance criteria must be verifiable by automated tests.

## 常见错误 / Common Mistakes

- 只写概念，没有写输入输出和验收。
- 把默认数字散落在多个实现文件。
- 失败时直接 throw，导致主循环无法恢复。
- 没有 replay case，后续重构容易破坏行为。

- Writing concepts without inputs, outputs, and acceptance criteria.
- Scattering default numbers across implementation files.
- Throwing on failure so the main loop cannot recover.
- Skipping replay cases, making future refactors risky.

## 本章总结 / Summary

本章的重点是把 `context` 层变成可实现、可测试、可恢复的工程边界。只要边界清楚，后续实现者就不需要靠猜。

The focus of this chapter is to turn the `context` layer into an implementable, testable, and recoverable engineering boundary. When boundaries are clear, implementers do not need to guess.

## 参考蓝图细节 / Reference Blueprint Details

以下内容保留原始架构蓝图中的细节、表格和代码片段，供实现时逐项对照。

The following section preserves the detailed tables, code snippets, and notes from the original architecture blueprint for implementation reference.

## 7.1 Context Window Numbers

Recommended defaults:

| Setting | Value |
|---|---:|
| Default context window | `200_000 tokens` |
| Large context window | `1_000_000 tokens` |
| Compact summary max output | `20_000 tokens` |
| Auto-compact buffer | `13_000 tokens` |
| Warning buffer | `20_000 tokens` |
| Error buffer | `20_000 tokens` |
| Manual compact reserve | `3_000 tokens` |
| Auto-compact failure circuit breaker | `3 failures` |

Formula:

```text
effectiveWindow = contextWindow - min(modelMaxOutputTokens, 20_000)
autoCompactThreshold = effectiveWindow - 13_000
blockingLimit = effectiveWindow - 3_000
warningThresholdWhenAutoCompactOn = autoCompactThreshold - 20_000
errorThresholdWhenAutoCompactOn = autoCompactThreshold - 20_000
warningThresholdWhenAutoCompactOff = effectiveWindow - 20_000
errorThresholdWhenAutoCompactOff = effectiveWindow - 20_000
```

Important detail:

The warning/error threshold is calculated from the active threshold. When auto-compact is enabled, the active threshold is `autoCompactThreshold`; when auto-compact is disabled, the active threshold is `effectiveWindow`.

Example for a 200k model with `modelMaxOutputTokens >= 20_000`:

```text
contextWindow = 200_000
summaryReserve = 20_000
effectiveWindow = 180_000
autoCompactThreshold = 167_000
blockingLimit = 177_000
warningThreshold = 147_000
errorThreshold = 147_000
```

Example for a 1M model with `modelMaxOutputTokens >= 20_000`:

```text
contextWindow = 1_000_000
summaryReserve = 20_000
effectiveWindow = 980_000
autoCompactThreshold = 967_000
blockingLimit = 977_000
warningThreshold = 947_000
errorThreshold = 947_000
```

Example for a model whose `modelMaxOutputTokens = 8_192`:

```text
contextWindow = 200_000
summaryReserve = 8_192
effectiveWindow = 191_808
autoCompactThreshold = 178_808
blockingLimit = 188_808
warningThreshold = 158_808
errorThreshold = 158_808
```

Recommended config shape:

```ts
type ContextBudgetConfig = {
  contextWindowTokens: number
  modelMaxOutputTokens: number
  compactSummaryReserveCapTokens: 20_000
  autoCompactBufferTokens: 13_000
  manualCompactReserveTokens: 3_000
  warningBufferTokens: 20_000
  errorBufferTokens: 20_000
  maxConsecutiveAutoCompactFailures: 3
}
```

Budget decision function:

```ts
function getContextDecision(inputTokens: number, cfg: ContextBudgetConfig) {
  const summaryReserve = Math.min(
    cfg.modelMaxOutputTokens,
    cfg.compactSummaryReserveCapTokens,
  )
  const effectiveWindow = cfg.contextWindowTokens - summaryReserve
  const autoCompactAt = effectiveWindow - cfg.autoCompactBufferTokens
  const blockAt = effectiveWindow - cfg.manualCompactReserveTokens
  const warnAt = autoCompactAt - cfg.warningBufferTokens

  if (inputTokens >= blockAt) return "block"
  if (inputTokens >= autoCompactAt) return "auto_compact"
  if (inputTokens >= warnAt) return "warn"
  return "continue"
}
```

Trigger order:

1. Persist or microcompact large tool results before counting full context.
2. Count estimated prompt tokens.
3. If `inputTokens >= blockingLimit`, block and ask for compact/clear unless reactive retry is already in progress.
4. If `inputTokens >= autoCompactThreshold`, run auto-compact unless disabled.
5. If API still rejects with prompt-too-long, run reactive compact and retry once.
6. If compaction fails 3 consecutive times, stop proactive compact for that session and surface a blocking error.

## 7.2 Model Output Token Defaults

Recommended table:

| Model family | Default output | Upper limit |
|---|---:|---:|
| Opus 4.6 | `64_000` | `128_000` |
| Sonnet 4.6 | `32_000` | `128_000` |
| Opus 4.5 / Sonnet 4 / Haiku 4 | `32_000` | `64_000` |
| Opus 4 / Opus 4.1 | `32_000` | `32_000` |
| Claude 3 Opus | `4_096` | `4_096` |
| Claude 3 Sonnet | `8_192` | `8_192` |
| Claude 3 Haiku | `4_096` | `4_096` |
| Claude 3.5 Sonnet / Haiku | `8_192` | `8_192` |
| Claude 3.7 Sonnet | `32_000` | `64_000` |
| Unknown modern model | `32_000` | `64_000` |

Optional optimization:

| Setting | Value |
|---|---:|
| Capped default output | `8_000 tokens` |
| Escalated retry output | `64_000 tokens` |
| Max output recovery turns | `3` |

## 7.3 Compaction Strategy

Use multiple compaction levels:

| Level | Trigger | Action |
|---|---|---|
| Tool result persistence | Tool result too large | Save to disk, insert preview |
| Microcompact | Old bulky tool results | Replace old result content with marker |
| Session memory | Long conversation | Extract durable notes |
| Full compact | Near context limit | Summarize conversation into new messages |
| Reactive compact | API says prompt too long | Compact and retry once |

Microcompact should target bulky tools:

- Read,
- Bash/PowerShell,
- Grep,
- Glob,
- WebSearch,
- WebFetch,
- Edit,
- Write.

For images/documents in rough token estimates, use `2_000 tokens` each.

## 7.4 Context Anti-Patterns

Avoid these implementation mistakes:

| Mistake | Concrete Consequence | Correct Rule |
|---|---|---|
| Counting only visible UI messages | Hidden tool results overflow provider context | Count provider-bound normalized messages |
| Keeping full Bash/Grep output inline | One command can consume 50k-200k tokens | Persist large results and inline preview |
| Compacting after API rejection only | User waits for failed call first | Proactive compact at `effectiveWindow - 13_000` |
| Summarizing file paths vaguely | Agent cannot resume edits safely | Preserve exact paths, commands, error text |
| Dropping tool-use pairs during compact | Provider rejects invalid transcript | Compact content, not pairing metadata |
| Letting subagents share parent history wholesale | Child wastes context and leaks instructions | Fork only necessary context and own sidechain |
