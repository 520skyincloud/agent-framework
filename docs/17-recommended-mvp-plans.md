# 17. Recommended MVP Plans

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
