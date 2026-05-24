> 中文标题：可观测性
>
> English title: Observability
>
> 中文导读：本章定义需要追踪的事件、事件 schema、关联 ID、生产测试矩阵、SLO 和远程部署限流建议。
>
> English guide: This chapter defines events to track, event schema, correlation IDs, production test matrix, SLOs, and remote deployment rate limits.

# 16. Observability

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
