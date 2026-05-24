# 25. SDK And UI Event Protocol

An Agent framework should expose one event stream that CLI, GUI, SDK, and remote clients can all consume.

## 25.1 SDK Event Union

```ts
type SDKEvent =
  | { type: "session_start"; sessionId: string; model: string }
  | { type: "user_message"; message: UserMessage }
  | { type: "assistant_delta"; messageId: string; text: string }
  | { type: "assistant_message"; message: AssistantMessage }
  | { type: "tool_use_start"; toolUseId: string; name: string; input: unknown }
  | { type: "tool_progress"; toolUseId: string; data: unknown }
  | { type: "tool_result"; toolUseId: string; ok: boolean; preview: string }
  | { type: "permission_request"; request: PermissionRequest }
  | { type: "task_update"; task: TaskState }
  | { type: "agent_update"; agent: AgentState }
  | { type: "compact_boundary"; metadata: CompactMetadata }
  | { type: "error"; error: RuntimeError }
  | { type: "done"; reason: TerminalReason }
```

## 25.2 Control Messages

```ts
type ControlMessage =
  | { type: "abort"; reason?: string }
  | { type: "permission_response"; requestId: string; decision: "allow" | "deny"; rule?: string }
  | { type: "task_stop"; taskId: string }
  | { type: "task_background"; taskId: string }
  | { type: "resume"; sessionId: string }
```

UI rules:

| Rule | Reason |
|---|---|
| Render assistant deltas separately from committed messages | Avoid duplicate text on retry |
| Show tool progress after `2_000 ms` | Short tools do not need visual noise |
| Permission prompt must include exact tool input summary | User needs concrete risk |
| Background tasks must be visible within `500 ms` | User must know work is running |
| Compact boundary should be visible but not verbose | User needs history marker |
| Large output should show preview and path | Avoid freezing UI |
