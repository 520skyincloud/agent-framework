> 中文标题：预算、速率限制与配额控制
>
> English title: Budget, Rate Limit, And Quota Control
>
> 中文导读：本章定义 token、成本、工具调用、Agent spawn、后台任务和存储预算，以及预算超限时如何返回模型可见消息。
>
> English guide: This chapter defines budgets for tokens, cost, tool calls, agent spawns, background tasks, and storage, plus model-visible budget-exceeded messages.

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
