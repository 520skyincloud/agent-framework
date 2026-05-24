> 中文标题：模型适配层与模型路由
>
> English title: Provider Adapter And Model Routing
>
> 中文导读：本章定义 ProviderAdapter、模型事件、重试策略、退避数字、stream watchdog、模型路由和不同调用类型的输出预算。
>
> English guide: This chapter defines ProviderAdapter, model events, retry policy, backoff numbers, stream watchdog, model routing, and output budgets for call types.

# 21. Provider Adapter And Model Routing

The model provider must be behind an adapter. Do not let provider-specific message shapes leak into the agent loop.

## 21.1 Provider Adapter Interface

```ts
type ProviderAdapter = {
  name: string
  supportsStreaming: boolean
  supportsToolUse: boolean
  supportsPromptCache: boolean
  supportsImages: boolean
  supportsPdf: boolean

  countTokens(input: ProviderRequest): Promise<TokenCount>
  stream(input: ProviderRequest, ctx: ProviderCallContext): AsyncGenerator<ModelEvent, ModelTerminal>
  complete(input: ProviderRequest, ctx: ProviderCallContext): Promise<ModelTerminal>
  normalizeError(error: unknown): ProviderError
}

type ProviderCallContext = {
  requestId: string
  sessionId: string
  turnId: string
  model: string
  fallbackModel?: string
  maxOutputTokens: number
  thinking?: ThinkingConfig
  signal: AbortSignal
  querySource: QuerySource
}
```

Normalized model events:

| Event | Required Fields |
|---|---|
| `message_start` | `requestId`, `model`, `usage?` |
| `content_delta` | `index`, `text` |
| `tool_use_delta` | `index`, `partialJson` |
| `tool_use_complete` | `id`, `name`, `input` |
| `usage_delta` | `inputTokens?`, `outputTokens?`, `cacheRead?`, `cacheCreate?` |
| `message_stop` | `stopReason`, `usage` |
| `error` | `code`, `retryable`, `providerStatus?` |

## 21.2 Retry Policy

Recommended defaults:

| Setting | Value |
|---|---:|
| Default max retries | `10` |
| Base delay | `500 ms` |
| Max normal delay | `32_000 ms` |
| Jitter | `0-25%` of computed delay |
| Consecutive 529 threshold | `3` |
| Persistent retry max backoff | `300_000 ms` |
| Persistent retry reset cap | `21_600_000 ms` |
| Persistent retry heartbeat | `30_000 ms` |
| Stream idle timeout | `90_000 ms` |
| Non-stream fallback timeout, local | `300_000 ms` |
| Non-stream fallback timeout, remote | `120_000 ms` |
| Max-token overflow safety buffer | `1_000 tokens` |
| Min adjusted output tokens | `3_000 tokens` |

Backoff formula:

```ts
function retryDelay(attempt: number, retryAfterMs?: number) {
  if (retryAfterMs !== undefined) return retryAfterMs
  const base = Math.min(500 * 2 ** (attempt - 1), 32_000)
  return base + Math.random() * 0.25 * base
}
```

Retry decision table:

| Error | Foreground Query | Background Query | Action |
|---|---|---|---|
| 429 | retry | usually fail fast unless persistent | respect `Retry-After` |
| 529 | retry up to policy | fail fast by default | avoid capacity amplification |
| 5xx | retry | retry if user-visible | exponential backoff |
| 401 | refresh auth once | refresh auth once | rebuild client |
| 403 revoked token | refresh auth once | refresh auth once | rebuild client |
| ECONNRESET/EPIPE | retry with fresh connection | retry with fresh connection | disable keep-alive if needed |
| prompt too long | reactive compact once | reactive compact once | retry compacted request |
| max output tokens | escalate up to `64_000` | escalate up to `64_000` | max `3` recovery turns |

## 21.3 Model Routing

Use a routing object instead of scattering model decisions:

```ts
type ModelRoute = {
  mainModel: string
  fallbackModel?: string
  compactModel?: string
  titleModel?: string
  classifierModel?: string
  verificationModel?: string
}
```

Concrete defaults:

| Call Type | Model Choice | Max Output |
|---|---|---:|
| main agent | user-selected | model default or capped `8_000` |
| max-output retry | same model | `64_000` |
| compact | compact model or main model | `20_000` |
| permission classifier | fast/cheap classifier model | `4_096` |
| title/summary | fast/cheap model | `4_096-8_192` |
| verification agent | same family as main or cheaper strong model | `32_000` |

Never route by string matching inside the agent loop. Resolve the route before `query()` starts.
