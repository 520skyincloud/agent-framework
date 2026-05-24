# 20. Design Rules Worth Keeping

1. Treat tools as typed, permissioned capabilities, not arbitrary functions.
2. Keep the agent loop small and explicit.
3. Make every side effect pass through permissions.
4. Preserve provider transcript invariants even on failure.
5. Put large outputs on disk, not in context.
6. Use read caches to protect edits from stale state.
7. Isolate subagents by default.
8. Make background work visible and stoppable.
9. Compact before the API rejects the request.
10. Persist enough to resume after process death.
