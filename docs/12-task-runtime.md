# 12. Task Runtime

A production agent needs background tasks.

Task types:

```ts
type Task =
  | LocalShellTask
  | LocalAgentTask
  | RemoteAgentTask
  | InProcessTeammateTask
  | MonitorTask
```

Required task fields:

| Field | Purpose |
|---|---|
| `taskId` | Stable ID |
| `type` | shell, agent, remote |
| `status` | running, completed, failed, stopped |
| `command` | Original action |
| `description` | UI label |
| `startedAt` | Timing |
| `completedAt?` | Timing |
| `outputPath?` | Large output |
| `agentId?` | Owner |
| `toolUseId?` | Parent tool call |
| `abortController?` | Stop task |

TaskOutput defaults:

| Setting | Value |
|---|---:|
| Wait timeout default | `30_000 ms` |
| Wait timeout max | `600_000 ms` |
| Tool result threshold | `100_000 chars` |
