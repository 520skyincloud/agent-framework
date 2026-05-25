# 推荐 MVP 路线 / Recommended MVP Plans

## 本章目标 / Goals

把单 Agent 和多 Agent 分成可验收的构建阶段。

Break single-agent and multi-agent work into phases with acceptance criteria.

完成本章后，读者应该知道这一层为什么存在、如何实现最小版本、哪些默认值不能随意改、以及如何验收。

After this chapter, the reader should know why this layer exists, how to implement the minimal version, which defaults must not be changed casually, and how to validate it.

## 核心概念 / Core Concepts

- MVP 应该减少复杂度，不减少安全性。
- 本章能力必须有清晰的输入、输出、失败语义和测试边界。
- 任何影响模型下一步行为的状态，都必须能被记录、恢复或回放。

- An MVP should reduce complexity, not safety.
- This capability must have clear input, output, failure semantics, and test boundaries.
- Any state that affects the model's next action must be recordable, recoverable, or replayable.

## 架构位置 / Where This Fits

本章位于 `mvp` 层。它不是孤立模块，而是和主循环、工具系统、上下文管理、持久化、权限或 SDK 事件流共同工作。实现时要明确本层是否拥有状态，是否会产生副作用，是否会改变下一轮模型输入。

This chapter belongs to the `mvp` layer. It is not an isolated module; it works with the main loop, tool system, context management, persistence, permissions, or SDK event stream. Implementation must make clear whether this layer owns state, produces side effects, or changes the next model input.

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

建议接口命名保持直接，例如 `mvpConfig`、`mvpState`、`mvpEvent`、`mvpResult`。如果这些类型变得过大，优先拆分所有权，而不是把所有字段塞进一个全局对象。

Use direct names such as `mvpConfig`, `mvpState`, `mvpEvent`, and `mvpResult`. If these types become too large, split ownership instead of putting every field into one global object.

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

本章的重点是把 `mvp` 层变成可实现、可测试、可恢复的工程边界。只要边界清楚，后续实现者就不需要靠猜。

The focus of this chapter is to turn the `mvp` layer into an implementable, testable, and recoverable engineering boundary. When boundaries are clear, implementers do not need to guess.

## 参考蓝图细节 / Reference Blueprint Details

以下内容保留原始架构蓝图中的细节、表格和代码片段，供实现时逐项对照。

The following section preserves the detailed tables, code snippets, and notes from the original architecture blueprint for implementation reference.

## 17.1 Single-Agent MVP

Build in four phases.

Phase 1: chat runtime.

| Item | Requirement | Acceptance Test |
|---|---|---|
| Message model | Normalize user/assistant/system/progress/tool result | Can replay transcript into provider format |
| Provider adapter | Streaming text response | UI receives partial assistant text |
| Query loop | Async generator | One user prompt produces one final assistant answer |
| Abort | AbortController | Ctrl+C stops streaming and persists abort reason |

Phase 2: read-only tools.

| Item | Requirement | Acceptance Test |
|---|---|---|
| Tool registry | Name/alias lookup | Unknown tool returns structured error |
| Tool schema | Zod or equivalent | Invalid JSON becomes tool_result error |
| Read | `file_path`, `offset`, `limit` | Large file requires slice |
| Grep | pattern + optional type/glob | Default returns at most `250` |
| Glob | pattern | Default returns at most `100` |
| Tool pairing | `tool_use_id` -> `tool_result` | Validator passes before next model call |

Phase 3: side effects.

| Item | Requirement | Acceptance Test |
|---|---|---|
| Permissions | allow/deny/ask | `Write` asks in default mode |
| Bash | timeout and progress | `sleep 5` shows progress by `2_000 ms` and respects timeout |
| Edit | requires prior read | Edit without Read fails |
| Write | workspace-bound | Write outside workspace asks or denies |
| Result persistence | large output to file | `yes | head -n 100000` does not inline full output |

Phase 4: durability.

| Item | Requirement | Acceptance Test |
|---|---|---|
| Transcript | append-only session log | Process restart can show prior turns |
| Tool output files | deterministic paths | Model receives preview + absolute path |
| Basic compact | full summarize near limit | 200k context compacts at `167_000` tokens |
| Usage accounting | tokens/cost/duration | Each model call has usage metadata |
| Resume repair | orphaned tool pairs fixed | Corrupted transcript can be repaired or blocked clearly |

MVP default numbers:

| Setting | Value |
|---|---:|
| Max tool concurrency | `4` initially, later `10` |
| Bash timeout | `120_000 ms` |
| Read output limit | `25_000 tokens` |
| Grep limit | `250` |
| Glob limit | `100` |
| Tool result threshold | `50_000 chars` |
| Auto-compact threshold | `context - 20_000 - 13_000` |

Definition of done for single-agent MVP:

1. Can answer normal chat without tools.
2. Can inspect a codebase with `Read`, `Grep`, `Glob`.
3. Can run shell commands with timeout and permission checks.
4. Can edit files only after reading them.
5. Can survive process restart with transcript intact.
6. Can handle one model-request failure, one tool failure, and one user abort without invalid transcript.
7. Can keep a 200k-token conversation below provider limit through persistence and compact.

## 17.2 Multi-Agent MVP

Add after single-agent is stable:

Phase 1: synchronous delegation.

| Item | Requirement | Acceptance Test |
|---|---|---|
| Agent definition loader | YAML/frontmatter/schema | Invalid `maxTurns` fails load |
| `Agent` tool | Spawns one child loop | Parent gets final text |
| `createSubagentContext` | Isolated messages/tools/permissions | Child cannot mutate parent state directly |
| Sidechain transcript | Separate log file | Child conversation can be inspected |

Phase 2: async agents.

| Item | Requirement | Acceptance Test |
|---|---|---|
| Background agent task | Return task handle immediately | Parent can continue |
| `TaskOutput` | Read status/output | Can fetch partial and final result |
| `TaskStop` | Stop child | Child aborts and marks task stopped |
| Progress summaries | Periodic status | UI shows at most `3` progress messages |

Phase 3: agent collaboration.

| Item | Requirement | Acceptance Test |
|---|---|---|
| Named registry | `name -> agentId` | Parent can send follow-up to live agent |
| Output file | Large result path | Parent reads output without bloating context |
| Verification agent | Read/Bash/browser capable | Verifier reports exact command results |
| Agent-specific tools | Tool allowlist | Plan agent cannot write files |

Phase 4: isolation.

| Item | Requirement | Acceptance Test |
|---|---|---|
| Worktree isolation | Separate cwd for writers | Two agents edit without clobbering |
| Permission bubbling | Foreground ask, background deny/notify | Async task never hangs invisibly |
| MCP per agent | Agent-specific server set | Child cleans up MCP server on exit |
| Resume metadata | Agent/task metadata persisted | Restart can show active/completed agents |

MVP multi-agent defaults:

| Setting | Value |
|---|---:|
| Default subagent maxTurns | `20` |
| Fork/deep worker maxTurns | `200` |
| Agent MCP wait | `30_000 ms` |
| MCP poll interval | `500 ms` |
| Async output wait default | `30_000 ms` |

Definition of done for multi-agent MVP:

1. Main agent can spawn a read-only Explore agent and receive a concise finding list.
2. Main agent can spawn an async Verify agent and continue the conversation immediately.
3. User can inspect and stop a background agent.
4. Sidechain transcript exists for every subagent.
5. Parent and child contexts have separate message arrays and abort behavior.
6. Permission asks never disappear into a background task.
7. Agent output larger than `100_000 chars` is stored as a file and summarized.
