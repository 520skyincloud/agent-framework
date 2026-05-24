# 4. Tool Protocol

## 4.1 Tool Interface

Every tool should implement the same contract:

```ts
type Tool<Input, Output, Progress = unknown> = {
  name: string
  aliases?: string[]
  searchHint?: string
  inputSchema: ZodSchema<Input>
  outputSchema?: ZodSchema<Output>

  prompt(ctx: ToolPromptContext): Promise<string>
  description(input: Input, ctx: ToolDescriptionContext): Promise<string>

  validateInput?(input: Input, ctx: ToolUseContext): Promise<ValidationResult>
  checkPermissions(input: Input, ctx: ToolUseContext): Promise<PermissionDecision>

  call(
    input: Input,
    ctx: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: (progress: Progress) => void
  ): Promise<ToolResult<Output>>

  isReadOnly(input: Input): boolean
  isConcurrencySafe(input: Input): boolean
  isDestructive?(input: Input): boolean
  interruptBehavior?(): "cancel" | "block"

  maxResultSizeChars: number
  mapToolResultToToolResultBlockParam(output: Output, toolUseId: string): ProviderToolResult
}
```

## 4.1.1 Tool Call Lifecycle

Every tool call must pass through the same ordered pipeline:

| Step | Function | Failure Output |
|---:|---|---|
| 1 | Find tool by provider name or alias | `tool_result` error: unknown tool |
| 2 | Parse input with `inputSchema` | `tool_result` error: invalid JSON/schema |
| 3 | Run `validateInput` | `tool_result` error: invalid domain input |
| 4 | Build human-readable `description` | fallback to tool name if formatter fails |
| 5 | Run permission pipeline | `tool_result` denial, not thrown exception |
| 6 | Check concurrency eligibility | schedule in safe batch or serial lane |
| 7 | Execute `call` with abort signal | normal output or error output |
| 8 | Persist large output if needed | preview + path |
| 9 | Map to provider `tool_result` block | must include original `tool_use_id` |
| 10 | Append to local transcript | emit progress and final result |

Minimal result shape:

```ts
type ToolResult<Output> =
  | {
      ok: true
      data: Output
      displayText?: string
      metadata?: Record<string, unknown>
    }
  | {
      ok: false
      error: string
      recoverable: boolean
      metadata?: Record<string, unknown>
    }
```

Concrete rule:

Do not throw tool errors across the agent loop boundary. Convert them into structured tool results so the model can recover.

## 4.2 Built-In Tools For A Coding Agent

Minimum set:

| Tool | Required | Notes |
|---|---|---|
| `Read` | yes | Read text, image, PDF, notebook |
| `Edit` | yes | Replace exact string in existing file |
| `Write` | yes | Create/overwrite file |
| `Bash` | yes | Shell command execution |
| `Grep` | yes | Fast content search |
| `Glob` | yes | Fast file discovery |
| `TodoWrite` | recommended | Plan/progress state for long tasks |
| `Agent` | optional first, required for multi-agent | Spawn subagents |
| `TaskOutput` | required if background tasks exist | Read async output |
| `TaskStop` | required if background tasks exist | Kill async tasks |
| `WebFetch` | optional | Fetch URL content |
| `WebSearch` | optional | Search web |
| `MCPTool` | optional | External tool bridge |

## 4.3 Tool Result Persistence

Do not blindly inline large tool results into context.

Recommended defaults copied from the studied system:

| Limit | Value |
|---|---:|
| Default per-tool result persistence threshold | `50_000 chars` |
| Aggregate tool results per user message | `200_000 chars` |
| Tool result token hard budget | `100_000 tokens` |
| Byte/token estimate | `4 bytes/token` |
| Persisted preview size | `2_000 bytes` |
| Bash inline threshold | `30_000 chars` |
| Grep inline threshold | `20_000 chars` |
| Bash persisted output max retained | `64 MB` |

When a result exceeds the threshold:

1. write full result to session tool-results directory,
2. send the model a preview,
3. include absolute file path,
4. allow the model to read the saved file if needed.

Example model-facing replacement:

```xml
<persisted-output>
Output too large (1.2 MB). Full output saved to: /path/to/tool-results/toolu_123.txt

Preview (first 2 KB):
...
</persisted-output>
```

## 4.4 Tool Output Classes

Classify every tool output before adding it to context:

| Class | Size | Context Behavior | Storage Behavior |
|---|---:|---|---|
| small | `< 20_000 chars` | inline full text | transcript only |
| medium | `20_000-50_000 chars` | inline unless tool has lower cap | transcript only |
| large | `> 50_000 chars` | preview only | save full output to file |
| huge | `> 100_000 tokens estimated` | error or forced persistence | save full output if possible |
| binary/media | any | provider media block or extracted text | cache original file |

Per-tool override examples:

| Tool | Suggested `maxResultSizeChars` |
|---|---:|
| `Bash` | `30_000` |
| `Grep` | `20_000` |
| `Glob` | `100_000` |
| `Agent` | `100_000` |
| `TaskOutput` | `100_000` |
| `Read` | `Infinity` or no persistence wrapper; it self-limits by tokens |
