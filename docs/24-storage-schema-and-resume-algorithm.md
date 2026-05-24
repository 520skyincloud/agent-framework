# 24. Storage Schema And Resume Algorithm

Persistence should be append-only where possible.

## 24.1 Transcript JSONL

Each line:

```ts
type TranscriptRecord = {
  version: 1
  sessionId: string
  message: Message
  parentUuid?: string
  writtenAt: string
  providerRequestId?: string
  compactionId?: string
}
```

Rules:

| Rule | Value |
|---|---|
| Write mode | append-only JSONL |
| Flush user message | before model call |
| Flush assistant | as complete message or safe chunks |
| Flush tool results | before next model call |
| Max JSONL line | `1 MB`; larger payloads must reference files |
| Message ID | UUID, never array index |

## 24.2 Tool Output Record

```ts
type ToolOutputRecord = {
  version: 1
  toolUseId: string
  toolName: string
  sessionId: string
  createdAt: string
  outputPath: string
  preview: string
  bytes: number
  sha256: string
  wasTruncatedForModel: boolean
}
```

## 24.3 Resume Algorithm

```text
1. Load metadata.json.
2. Replay transcript.jsonl in order.
3. Validate message UUID uniqueness.
4. Validate tool-use/tool-result pairs.
5. Load task metadata and mark stale running tasks as unknown/stopped.
6. Load agent metadata and sidechain paths.
7. Verify referenced tool output files exist.
8. Insert repair system messages for missing files or orphaned pairs.
9. Recompute token usage estimate.
10. If over blocking limit, require compact before next user turn.
```

Repair table:

| Problem | Repair |
|---|---|
| duplicate UUID | keep first, tombstone later duplicate |
| missing tool result | synthesize error result |
| orphan tool result | omit from provider context, keep local audit |
| missing tool output file | replace with error marker |
| stale running task | mark `unknown` and require user action |
| deleted worktree | mark agent blocked |
