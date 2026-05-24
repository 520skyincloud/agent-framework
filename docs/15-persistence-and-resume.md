> 中文标题：持久化与恢复
>
> English title: Persistence And Resume
>
> 中文导读：本章说明主 transcript、sidechain transcript、工具输出、后台任务、Agent metadata、文件历史如何写入和恢复。
>
> English guide: This chapter explains how to persist and resume main transcripts, sidechain transcripts, tool outputs, background tasks, agent metadata, and file history.

# 15. Persistence And Resume

Persist:

| Artifact | Why |
|---|---|
| Main transcript | Resume conversation |
| Sidechain transcripts | Resume/debug subagents |
| Tool result files | Avoid context bloat |
| Background task output | Async result retrieval |
| Agent metadata | Restore agent type/cwd/worktree |
| Content replacement records | Prompt cache stability on resume |
| File history snapshots | Undo/review |

Write order:

1. persist accepted user message,
2. stream assistant and progress messages,
3. persist tool results,
4. persist compact boundaries,
5. flush session storage on important transitions.

Resume must repair:

- orphaned tool uses,
- orphaned tool results,
- missing sidechain metadata,
- deleted worktree paths,
- stale pending permissions.
