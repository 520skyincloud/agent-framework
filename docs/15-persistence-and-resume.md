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
