# 28. Budget, Rate Limit, And Quota Control

Budget control must happen before model and tool calls, not only after billing arrives.

## 28.1 Budget Types

```ts
type BudgetState = {
  maxUsdPerSession?: number
  maxUsdPerTurn?: number
  maxInputTokensPerTurn: number
  maxOutputTokensPerTurn: number
  maxToolCallsPerTurn: number
  maxAgentSpawnsPerTurn: number
  maxWallClockMsPerTurn: number
  maxBackgroundTasks: number
}
```

Recommended starting defaults:

| Budget | Value |
|---|---:|
| Max tool calls per turn | `30` |
| Max agent spawns per turn | `3` |
| Max wall-clock per foreground turn | `900_000 ms` |
| Max background tasks per user | `8` |
| Max concurrent subagents | `4` |
| Max persisted output per session | `1 GB` |
| Stop after repeated same tool error | `3` |

## 28.2 Budget Enforcement Points

| Point | Check |
|---|---|
| Before model call | token and USD estimate |
| During stream | max output and wall clock |
| Before tool batch | tool count and risk |
| Before subagent launch | spawn count and concurrent agent count |
| Before background task | task limit |
| Before persistence | session storage limit |

When a budget is exceeded, return a normal model-visible message:

```text
Budget exceeded: maxAgentSpawnsPerTurn=3. Continue with existing agents or ask the user to approve more.
```
