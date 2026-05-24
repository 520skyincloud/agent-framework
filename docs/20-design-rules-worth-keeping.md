> 中文标题：值得保留的设计规则
>
> English title: Design Rules Worth Keeping
>
> 中文导读：本章总结十条设计原则，例如工具必须 typed/permissioned、大输出落盘、子 Agent 默认隔离、后台任务可见可停等。
>
> English guide: This chapter summarizes ten design principles, such as typed/permissioned tools, disk persistence for large output, isolated subagents, and visible/stoppable background work.

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
