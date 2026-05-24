> 中文标题：多 Agent 冲突处理
>
> English title: Multi-Agent Conflict Handling
>
> 中文导读：本章说明 FileLease、读写冲突策略、worktree 隔离和多 Agent 工作合并协议。
>
> English guide: This chapter explains FileLease, read/write conflict policy, worktree isolation, and multi-agent merge protocol.

# 29. Multi-Agent Conflict Handling

Spawning multiple agents is easy. Merging their work is the hard part.

## 29.1 File Ownership

Track ownership per task:

```ts
type FileLease = {
  path: string
  ownerAgentId: string
  mode: "read" | "write"
  expiresAt: string
  reason: string
}
```

Defaults:

| Setting | Value |
|---|---:|
| Write lease TTL | `30 minutes` |
| Read lease TTL | `10 minutes` |
| Max files per agent lease | `50` |
| Conflict check interval | before every write |

Conflict policy:

| Conflict | Action |
|---|---|
| two agents read same file | allow |
| one agent writes file another only read | warn parent |
| two agents want to write same file | block second or require separate worktree |
| agent writes file changed by user | fail stale edit |
| worktree merge conflict | report exact files to parent |

## 29.2 Worktree Strategy

Use worktrees for parallel writing agents:

```text
main workspace
  parent agent reads and coordinates

worktree/agent_<id>
  implementer agent edits files
  verifier runs tests
  parent merges or rejects diff
```

Merge protocol:

1. Agent finishes with patch summary.
2. Parent inspects changed files.
3. Verifier runs tests in worktree.
4. Parent applies patch to main workspace only if clean.
5. If conflict, parent asks user or creates conflict report.
