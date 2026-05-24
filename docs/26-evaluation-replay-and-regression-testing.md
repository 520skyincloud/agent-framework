# 26. Evaluation, Replay, And Regression Testing

You cannot trust an Agent framework without replay tests.

## 26.1 Test Harness Layers

| Layer | What To Mock | What To Assert |
|---|---|---|
| Unit | tool input/output | schema, permissions, result format |
| Loop | provider event stream | pairing, retries, abort |
| Storage | JSONL and output files | resume repairs |
| Context | token counts and omissions | thresholds and compact triggers |
| Multi-agent | child event stream | isolation and task state |
| End-to-end | fake repo + fake model | final file diff and transcript |

## 26.2 Golden Replay Format

```ts
type ReplayCase = {
  name: string
  initialFiles: Record<string, string>
  userMessages: string[]
  modelScript: ModelEvent[][]
  expectedToolCalls: Array<{ name: string; input: unknown }>
  expectedFinalFiles?: Record<string, string>
  expectedTerminalReason: TerminalReason
  maxTurns: number
}
```

Minimum replay cases:

| Case | Purpose |
|---|---|
| `read_then_edit` | edit requires prior read |
| `parallel_reads` | safe tools run concurrently but return ordered |
| `tool_failure_recovery` | failed tool becomes model-visible result |
| `abort_during_bash` | transcript remains valid |
| `prompt_too_long_compact` | compact triggers at exact threshold |
| `missing_tool_result_resume` | resume repair works |
| `async_agent_finishes` | task output can be retrieved |
| `permission_denied_alternative` | model can choose safer path |

## 26.3 Quality Metrics

Track these over evals:

| Metric | Target |
|---|---:|
| Invalid provider transcript | `0` |
| Unhandled tool exception | `0` |
| Permission bypass | `0` |
| Resume success | `> 99%` |
| Tool result persistence correctness | `100%` |
| Repeated compact failure loop | `0` |
| Median turn latency regression | `< 10%` week over week |
