# 22. Prompt And Context Assembly

The framework needs a deterministic prompt pipeline. A vague "build context" step is not enough.

## 22.1 Prompt Assembly Order

Use this exact order:

1. Base system prompt.
2. Product/runtime rules.
3. Security and permission rules.
4. Tool-use rules.
5. Output style rules.
6. Agent-specific instructions.
7. Project instructions from files such as `CLAUDE.md` or `.agent.md`.
8. User profile/preferences.
9. Session memory.
10. Current task state/todos.
11. Tool schemas.
12. Recent conversation messages.
13. Attachments and media.
14. Synthetic repair messages if needed.

Budget allocation for a 200k context:

| Segment | Budget |
|---|---:|
| System + runtime rules | `8_000 tokens` |
| Tool schemas | `20_000 tokens` initially, less with dynamic loading |
| Project instructions | `10_000 tokens` |
| Session memory | `12_000 tokens` |
| Recent conversation | up to remaining budget |
| Media estimate | `2_000 tokens` each |
| Compact reserve | `20_000 tokens` |
| Auto-compact buffer | `13_000 tokens` |

## 22.2 Prompt Object

```ts
type PromptBundle = {
  providerMessages: ProviderMessage[]
  system: string | ContentBlock[]
  toolSchemas: ProviderToolSchema[]
  tokenEstimate: number
  cacheBreakpoints: CacheBreakpoint[]
  omitted: OmittedContextRecord[]
}

type OmittedContextRecord = {
  reason: "budget" | "deferred_tool" | "persisted_output" | "compacted" | "permission"
  itemId: string
  tokenEstimate: number
  replacementText?: string
}
```

Rules:

| Rule | Why |
|---|---|
| Prompt assembly must be pure | Allows replay and tests |
| Every omission must be recorded | Makes debugging possible |
| Tool schemas must be sorted stably | Protects prompt cache |
| Dynamic tools must have search hints | Model can discover them |
| Project instructions must preserve path | User can inspect source |
| Session memory must be structured | Prevents vague summaries |

## 22.3 Context Selection Algorithm

```text
1. Always include system prompt and mandatory tool schemas.
2. Include latest compact summary if present.
3. Include session memory if initialized.
4. Include project instructions within budget.
5. Include recent messages from newest to oldest.
6. Replace large tool results with persisted-output markers.
7. Drop or defer optional tool schemas if over budget.
8. If still over budget, trigger compact.
9. If compact already failed 3 times, block.
```

Important invariant:

Do not drop only one side of a tool-use/tool-result pair. If old content must be removed, replace the result content with a marker but preserve IDs.
