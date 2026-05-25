# 可观测性 / Observability

## 本章目标 / Goals

定义事件、指标、关联 ID、SLO 和生产测试矩阵。

Define events, metrics, correlation IDs, SLOs, and production test matrix.

完成本章后，读者应该知道这一层为什么存在、如何实现最小版本、哪些默认值不能随意改、以及如何验收。

After this chapter, the reader should know why this layer exists, how to implement the minimal version, which defaults must not be changed casually, and how to validate it.

## 核心概念 / Core Concepts

- 可观测性要能解释慢、贵、错。
- 本章能力必须有清晰的输入、输出、失败语义和测试边界。
- 任何影响模型下一步行为的状态，都必须能被记录、恢复或回放。

- Observability must explain slow, expensive, and failed turns.
- This capability must have clear input, output, failure semantics, and test boundaries.
- Any state that affects the model's next action must be recordable, recoverable, or replayable.

## 架构位置 / Where This Fits

本章位于 `observability` 层。它不是孤立模块，而是和主循环、工具系统、上下文管理、持久化、权限或 SDK 事件流共同工作。实现时要明确本层是否拥有状态，是否会产生副作用，是否会改变下一轮模型输入。

This chapter belongs to the `observability` layer. It is not an isolated module; it works with the main loop, tool system, context management, persistence, permissions, or SDK event stream. Implementation must make clear whether this layer owns state, produces side effects, or changes the next model input.

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

建议接口命名保持直接，例如 `observabilityConfig`、`observabilityState`、`observabilityEvent`、`observabilityResult`。如果这些类型变得过大，优先拆分所有权，而不是把所有字段塞进一个全局对象。

Use direct names such as `observabilityConfig`, `observabilityState`, `observabilityEvent`, and `observabilityResult`. If these types become too large, split ownership instead of putting every field into one global object.

## 默认值与关键数字 / Defaults And Numbers

| 配置 / Config | 默认值 / Default | 说明 / Notes |
|---|---:|---|
| `configSource` | `.agent/config.json` | 实现时显式引用，不要隐藏在业务逻辑中。 / Reference explicitly in implementation; do not hide it in business logic. |
| `owner` | `explicit module` | 实现时显式引用，不要隐藏在业务逻辑中。 / Reference explicitly in implementation; do not hide it in business logic. |
| `testMode` | `replay case required` | 实现时显式引用，不要隐藏在业务逻辑中。 / Reference explicitly in implementation; do not hide it in business logic. |

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

本章的重点是把 `observability` 层变成可实现、可测试、可恢复的工程边界。只要边界清楚，后续实现者就不需要靠猜。

The focus of this chapter is to turn the `observability` layer into an implementable, testable, and recoverable engineering boundary. When boundaries are clear, implementers do not need to guess.

## 参考蓝图细节 / Reference Blueprint Details

以下内容保留原始架构蓝图中的细节、表格和代码片段，供实现时逐项对照。

The following section preserves the detailed tables, code snippets, and notes from the original architecture blueprint for implementation reference.

Track at least:

| Event | Fields |
|---|---|
| model request start/end | model, duration, ttft, request ID |
| token usage | input, output, cache read, cache create |
| tool use start/end | tool name, tool ID, duration, error |
| permission decision | allow/deny/ask, source, rule |
| compaction | pre tokens, post tokens, saved tokens |
| agent spawn | agent type, async, model, tools |
| task lifecycle | task ID, status, duration |
| API fallback | original model, fallback model |
| prompt-too-long recovery | compact/retry/fail |

Keep analytics sanitized:

| Limit | Value |
|---|---:|
| Tool input JSON chars | `4 KB` |
| Collection items | `20` |
| Object depth | `2` |
| File extension length | `10` |

## 16.1 Minimum Event Schema

Use one envelope for all runtime events:

```ts
type RuntimeEvent = {
  eventId: string
  sessionId: string
  conversationId?: string
  agentId?: string
  parentAgentId?: string
  turnId?: string
  timestamp: string
  type: string
  durationMs?: number
  ok?: boolean
  errorCode?: string
  metadata?: Record<string, string | number | boolean | null>
}
```

Required correlation IDs:

| ID | Scope |
|---|---|
| `sessionId` | Process/user session |
| `conversationId` | Conversation transcript |
| `turnId` | One user -> model/tools cycle |
| `toolUseId` | Provider tool-use pair |
| `taskId` | Background shell/agent task |
| `agentId` | Main or subagent |
| `requestId` | Provider API request |

## 16.2 Production Test Matrix

Before calling the framework production-ready, test these exact cases:

| Area | Test Case | Expected Result |
|---|---|---|
| Transcript | Assistant emits 2 tool calls; second tool fails | Both tool calls get matching results |
| Abort | User aborts during Bash | Bash cancelled or backgrounded according to policy; transcript valid |
| Retry | Provider disconnects before content | Retry without duplicate user message |
| Max output | Model hits max output 3 times | Escalates up to `64_000`, then stops with explicit error |
| Prompt too long | 200k context reaches `167_000` | Auto-compact before provider rejection |
| Compact failure | Compact fails 3 consecutive times | Circuit breaker blocks further auto attempts |
| Tool persistence | Tool returns `75_000 chars` | Full result saved, preview sent |
| Aggregate results | 10 parallel tools return `40_000 chars` each | Largest outputs persisted until message under `200_000 chars` |
| Read cap | Read file over `256 KB` without range | Tool error asks for offset/limit |
| Edit stale file | File changed after Read | Edit fails before writing |
| Permission | `rm -rf` in default mode | Ask or deny; never auto-allow |
| Plan mode | Agent tries `Write` | Deny with tool result |
| Subagent isolation | Child edits parent state | Impossible by type/API |
| Async agent | Background agent requests permission | Deny/notify; does not hang |
| Resume | Process dies after user message before model response | Resume shows user message and can continue |
| Orphan repair | Persisted assistant has tool_use but no result | Validator synthesizes or blocks clearly |
| MCP failure | MCP server fails to connect after `30_000 ms` | Agent proceeds without that tool or reports unavailable |
| Web redirect | Redirect changes protocol/port | Block unsafe redirect |

## 16.3 Service-Level Defaults

Recommended SLOs for a local/desktop coding agent:

| Metric | Target |
|---|---:|
| Time to first token | `< 3_000 ms` for warm provider path |
| Tool permission prompt latency | `< 300 ms` after tool_use parsed |
| Read/Grep/Glob simple tool latency | `< 1_000 ms` on medium repos |
| Transcript append latency | `< 50 ms` local disk |
| Auto-compact success rate | `> 99%` when under provider hard limit |
| Invalid transcript sent to provider | `0` |
| Background task visible in UI | `< 500 ms` after launch |

For remote/product deployments, add rate limits per user:

| Limit | Suggested Starting Value |
|---|---:|
| Concurrent foreground sessions | `1` per conversation |
| Concurrent background tasks | `8` per user |
| Concurrent subagents | `4` per conversation |
| Max active MCP servers | `16` per session |
| Max persisted tool output per session | `1 GB` |
| Transcript retention | product policy, but never in provider context by default |
