> 中文标题：工具执行编排
>
> English title: Tool Execution Orchestration
>
> 中文导读：本章说明工具如何并发、如何分批、哪些工具必须串行、streaming tool execution 的边界，以及工具成功后如何修改上下文状态。
>
> English guide: This chapter explains tool concurrency, batching, which tools must run serially, streaming tool execution boundaries, and context modifiers after tool success.

# 5. Tool Execution Orchestration

## 5.1 Concurrency

Recommended default:

| Setting | Value |
|---|---:|
| Max concurrent tool calls | `10` |

Only run tools concurrently when `tool.isConcurrencySafe(input)` returns true.

Practical rule:

| Tool kind | Concurrent? |
|---|---|
| Read/search/list | yes |
| File edit/write | no |
| Shell command | only if classified read-only |
| MCP unknown side effects | no by default |
| Agent spawn | usually no for sync, yes for async |

Execution algorithm:

1. partition consecutive tool calls into batches,
2. consecutive safe tools run in parallel,
3. unsafe tools run serially,
4. apply context modifiers only after a safe batch completes,
5. preserve tool result ordering when sending back to the model.

Batching example:

```text
Input order:
  1 Read(a.ts)          safe
  2 Grep("foo")         safe
  3 Edit(a.ts)          unsafe
  4 Read(b.ts)          safe
  5 Bash("git diff")    safe, read-only
  6 Bash("npm test")    unsafe or long-running

Execution:
  Batch A parallel: 1, 2
  Serial: 3
  Batch B parallel: 4, 5
  Serial: 6

Provider result order:
  1, 2, 3, 4, 5, 6
```

Concurrency safety rules:

| Tool | `isReadOnly` | `isConcurrencySafe` | Notes |
|---|---:|---:|---|
| `Read` | yes | yes | Use file read cache |
| `Grep` | yes | yes | Limit output |
| `Glob` | yes | yes | Limit results |
| `WebFetch` | yes | yes | Domain/redirect policy still applies |
| `Bash(git status)` | yes | yes | Only if classifier says read-only |
| `Bash(npm test)` | no | no | May write caches, logs, snapshots |
| `Edit` | no | no | Must validate stale read |
| `Write` | no | no | Overwrite risk |
| `Agent(sync)` | usually no | no | Child may use tools |
| `Agent(async)` | no | yes as task launch | Parent gets task handle, not final result |
| Unknown MCP | unknown | no | Require explicit capability metadata |

## 5.2 Streaming Tool Execution

Advanced optimization:

Start executing a tool as soon as its complete `tool_use` block streams from the model, instead of waiting for the whole assistant message to finish.

Rules:

| Rule | Reason |
|---|---|
| Buffer results until order is safe | Provider expects coherent transcript |
| Cancel sibling shell commands if a Bash command errors | Avoid useless dependent work |
| On model fallback, discard in-flight results | Tool IDs from failed stream are invalid |
| On abort, synthesize tool results | Preserve pairing invariant |

This is not required for MVP. Add it after normal serial/batched execution works.

## 5.3 Context Modifiers

Some tools need to mutate agent-local state after success:

| Modifier | Tool | Purpose |
|---|---|---|
| `readFileState.add(path, hash, content)` | `Read` | Allows later `Edit` to prove file was read |
| `readFileState.update(path, newHash)` | `Edit` / `Write` | Prevents stale edits after mutation |
| `taskStore.add(task)` | `Bash(run_in_background)` / `Agent(background)` | Makes task visible and stoppable |
| `todoStore.replace(list)` | `TodoWrite` | UI progress state |
| `agentRegistry.add(name, agentId)` | `Agent` | Enables later named messages |

Do not apply context modifiers from concurrently running tools until the whole safe batch finishes. Otherwise a fast tool can change state that a slower sibling did not see when it started.
