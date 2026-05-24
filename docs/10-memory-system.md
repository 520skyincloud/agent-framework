# 10. Memory System

Use three memory layers:

| Layer | Lifetime | Example |
|---|---|---|
| Message history | Current session | Raw conversation |
| File read cache | Current session/subagent | Last read file contents and timestamps |
| Session memory | Long-running session | Structured notes file |

## 10.1 Session Memory Defaults

| Setting | Value |
|---|---:|
| Initialize after | `10_000 tokens` |
| Update after token growth | `5_000 tokens` |
| Update after tool calls | `3 tool calls` |
| Wait for extraction | `15_000 ms` |
| Treat extraction as stale | `60_000 ms` |
| Total session memory budget | `12_000 tokens` |
| Section budget | `2_000 tokens` |

Recommended session memory sections:

- Session Title,
- Current State,
- Task specification,
- Files and Functions,
- Workflow,
- Errors & Corrections,
- Codebase and System Documentation,
- Learnings,
- Key results,
- Worklog.

Important rule:

Session memory should preserve exact file paths, commands, decisions, and errors. It should not become a vague summary.
