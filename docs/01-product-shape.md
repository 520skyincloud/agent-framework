# 1. Product Shape

An agent system is not just an LLM wrapper. It is a runtime that repeatedly:

1. accepts user input,
2. builds context,
3. asks a model for the next action,
4. executes tool calls,
5. feeds tool results back to the model,
6. persists state,
7. stops, continues, compacts, or delegates.

The minimum useful system has these parts:

| Layer | Responsibility |
|---|---|
| Entry/UI | CLI, web UI, desktop UI, SDK, API endpoint |
| Session engine | One conversation lifecycle, mutable message history, abort handling |
| Agent loop | Model call -> tool use -> tool result -> model call |
| Tool runtime | Tool registry, schema validation, permissions, execution, result formatting |
| Context manager | System prompt, user/project context, token budgets, compaction |
| Permission system | Allow/deny/ask rules, per-tool safety checks, user approval |
| Memory | Short-term message history, file read cache, long-term/session memory |
| Task runtime | Background shell tasks, background agents, progress polling |
| Multi-agent layer | Subagent spawning, isolation, shared outputs, team messaging |
| Persistence | Transcript, tool outputs, sidechain agent logs, resumability |
| Observability | Events, tracing, debug logs, usage/cost/token accounting |
