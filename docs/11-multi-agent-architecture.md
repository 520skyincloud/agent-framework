# 11. Multi-Agent Architecture

## 11.1 Subagent Types

Recommended built-ins:

| Agent | Tools | Purpose |
|---|---|---|
| General | Read/Edit/Bash/Search | General delegated work |
| Explore | Read/Search only | Investigate codebase |
| Plan | Read/Search only | Produce implementation plan |
| Verify | Read/Bash/Search | Validate completed work |
| Specialist | Custom | Domain-specific agent |

Agent definition schema:

```yaml
name: Explore
description: Investigate code without editing
tools:
  - Read
  - Grep
  - Glob
  - Bash
permissionMode: plan
maxTurns: 20
model: sonnet
background: false
mcpServers:
  - github
skills:
  - code-review
```

## 11.2 Subagent Isolation

Each subagent should get:

| State | Default |
|---|---|
| `agentId` | new unique ID |
| messages | own initial prompt plus optional forked context |
| readFileState | cloned or fresh |
| abortController | child or independent if async |
| setAppState | no-op unless interactive |
| permission mode | agent-specific |
| tools | filtered for the agent |
| transcript | sidechain log |
| cwd | inherited, explicit override, or worktree |

Async subagents should avoid permission prompts unless they can bubble approval to the main thread.

## 11.3 Multi-Agent Limits

Recommended concrete defaults:

| Setting | Value |
|---|---:|
| Fork subagent maxTurns | `200` |
| Agent MCP wait max | `30_000 ms` |
| Agent MCP polling interval | `500 ms` |
| Agent progress hint threshold | `2_000 ms` |
| Agent UI progress messages shown | `3` |

## 11.4 Agent Communication

For real multi-agent systems, avoid only returning final strings.

Recommended channels:

| Channel | Use |
|---|---|
| Parent result | Final answer from subagent |
| Task notification | Async completion notification |
| Agent name registry | Send message to named live agent |
| Shared task store | List/stop/read background jobs |
| Sidechain transcript | Resume/debug subagent |
| Output file | Large async result |

## 11.5 Parent-Child Contract

The parent agent should never share its mutable state object directly with a subagent. It should create an explicit child context:

```ts
type SubagentLaunchRequest = {
  parentAgentId?: string
  prompt: string
  agentType: string
  model?: string
  maxTurns: number
  background: boolean
  cwd: string
  allowedTools: string[]
  permissionMode: PermissionMode
  forkContextMessages?: Message[]
  inheritedFiles?: string[]
}

type SubagentLaunchResult =
  | { mode: "sync"; agentId: string; finalText: string; transcriptPath: string }
  | { mode: "async"; agentId: string; taskId: string; outputPath: string }
```

Launch rules:

| Case | Rule |
|---|---|
| Small investigation | Use sync subagent with `maxTurns: 20` |
| Long-running verification | Use async subagent and return `taskId` immediately |
| Potential file edits | Give a separate worktree or strict file scope |
| Read-only planning | `permissionMode: plan`, tools `Read/Grep/Glob/Bash(read-only)` |
| External MCP needed | Wait up to `30_000 ms`, poll every `500 ms` |
| Parent abort | Cancel sync child; async child can continue only if explicitly backgrounded |

## 11.6 Multi-Agent Topologies

Choose one topology intentionally:

| Topology | Best For | Avoid When |
|---|---|---|
| Parent delegates to one specialist | Simple codebase investigation | Many independent files |
| Parent spawns N explore agents | Large repo exploration | Shared write operations |
| Planner -> implementer -> verifier | Coding tasks with clear success criteria | User wants fast one-shot answer |
| Background teammate agents | Long tasks, separate workstreams | Need interactive permission prompts |
| Named live agents | Ongoing collaboration | You cannot persist sidechain transcripts |

Recommended first multi-agent product:

```text
Main Agent
  - keeps user conversation
  - owns permissions
  - decides task split
  - merges final answer

Explore Agent
  - read-only
  - maxTurns = 20
  - returns file map and findings

Verify Agent
  - read + bash
  - background = true
  - validates tests/build/browser

TaskOutput
  - lets main agent fetch async verifier result
```

## 11.7 Permission Bubbling

Async agents should not block forever on prompts nobody can see.

Use this policy:

| Agent Mode | Permission Ask Behavior |
|---|---|
| Sync foreground subagent | Bubble ask to parent UI |
| Async background subagent | Auto-deny ask unless configured to notify parent |
| Read-only agent | Deny writes without asking |
| Worktree-isolated implementer | Allow scoped edits, ask for shell/destructive |
| Remote teammate | Require explicit allowlist before launch |

If permission is denied, return it as a normal tool result inside the child transcript, then let the subagent summarize the blockage to the parent.
