# 7. Context Management

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
