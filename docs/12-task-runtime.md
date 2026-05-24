> 中文标题：任务运行时
>
> English title: Task Runtime
>
> 中文导读：本章定义后台任务类型、任务字段、TaskOutput 默认值，以及 shell/agent/remote/monitor 任务如何被追踪。
>
> English guide: This chapter defines background task types, required task fields, TaskOutput defaults, and how shell, agent, remote, and monitor tasks are tracked.

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
