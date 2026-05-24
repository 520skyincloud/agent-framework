> 中文标题：配置系统与目录结构
>
> English title: Configuration And Directory Layout
>
> 中文导读：本章说明配置优先级、项目/用户配置文件、运行时目录结构和 AgentRuntimeConfig schema。
>
> English guide: This chapter explains config precedence, project/user config files, runtime directory layout, and AgentRuntimeConfig schema.

# 23. Configuration And Directory Layout

Agent frameworks become unmaintainable when constants live inside random modules.

## 23.1 Config Precedence

Use this precedence:

```text
CLI flag > environment variable > project config > user config > organization config > built-in default
```

Concrete config files:

| File | Scope |
|---|---|
| `.agent/config.json` | project-local runtime config |
| `.agent/agents/*.md` | project-local agent definitions |
| `.agent/permissions.json` | project-local permission rules |
| `.agent/mcp.json` | project-local MCP servers |
| `~/.agent/config.json` | user defaults |
| `~/.agent/permissions.json` | user permission rules |
| `~/.agent/sessions/` | transcripts and session metadata |

## 23.2 Runtime Directory Layout

Recommended session layout:

```text
~/.agent/
  config.json
  permissions.json
  sessions/
    2026-05-24/
      session_<id>/
        transcript.jsonl
        metadata.json
        tool-results/
          toolu_<id>.txt
        tasks/
          task_<id>.json
          task_<id>.out
        agents/
          agent_<id>/
            transcript.jsonl
            metadata.json
        file-history/
          <hash>.patch
        media/
          image_<id>.png
```

## 23.3 Typed Config Schema

```ts
type AgentRuntimeConfig = {
  model: ModelRoute
  context: ContextBudgetConfig
  tools: {
    maxConcurrency: number
    resultPersistChars: number
    perMessageResultChars: number
  }
  bash: {
    defaultTimeoutMs: number
    maxTimeoutMs: number
    progressAfterMs: number
    autoBackgroundAfterMs: number
  }
  permissions: {
    mode: PermissionMode
    allow: string[]
    deny: string[]
  }
  storage: {
    rootDir: string
    maxSessionBytes: number
    maxToolOutputBytes: number
  }
  agents: {
    defaultMaxTurns: number
    forkMaxTurns: number
    maxConcurrentSubagents: number
  }
}
```

Default starting values:

| Field | Value |
|---|---:|
| `tools.maxConcurrency` | `10` |
| `storage.maxSessionBytes` | `1 GB` |
| `agents.maxConcurrentSubagents` | `4` |
| `bash.defaultTimeoutMs` | `120_000` |
| `bash.maxTimeoutMs` | `600_000` |
