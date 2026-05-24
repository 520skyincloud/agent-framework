> 中文标题：会话引擎
>
> English title: Session Engine
>
> 中文导读：本章说明一个会话应该持有哪些状态，以及 QueryEngine 这类对象应该如何接收用户输入、处理中断、保存历史和输出事件。
>
> English guide: This chapter explains what state a session owns and how a QueryEngine-style object should accept user input, handle aborts, save history, and emit events.

# 3. Session Engine

The session engine owns one conversation.

Recommended responsibilities:

| Responsibility | Details |
|---|---|
| Mutable history | Current messages, compacted messages, attachments |
| Abort controller | User interrupt, timeout, child task cancellation |
| File read state | Tracks files read before editing |
| Permission context | Current permission mode and rules |
| Tool list | Built-ins plus MCP/plugin tools |
| Model config | main model, fallback model, thinking/effort |
| Persistence | transcript write before and after model response |
| Usage | token usage, cost, API duration |

Recommended public API:

```ts
class QueryEngine {
  constructor(config: QueryEngineConfig)

  submitMessage(
    prompt: string | ContentBlock[],
    options?: { uuid?: string; isMeta?: boolean }
  ): AsyncGenerator<SDKMessage>

  abort(reason?: string): void
}
```

Persist the user message before calling the model. If the process dies mid-request, resume should still know what the user asked.
