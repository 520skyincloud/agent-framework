# 6. Permissions And Safety

## 6.1 Permission Modes

Recommended modes:

| Mode | Behavior |
|---|---|
| `default` | Ask for writes/destructive/open-world actions |
| `acceptEdits` | Allow safe file edits in workspace, ask for shell/destructive |
| `plan` | Read/search only, no writes until exit |
| `auto` | Classifier decides allow/deny for ask cases |
| `dontAsk` | Convert ask to deny |
| `bypassPermissions` | Allow everything, only for trusted local use |

## 6.2 Permission Decision Pipeline

For every tool call:

1. parse input with schema,
2. run tool-specific `validateInput`,
3. run pre-tool hooks,
4. check deny rules,
5. check allow rules,
6. run tool-specific `checkPermissions`,
7. if ask:
   - run permission hooks,
   - run classifier if in auto mode,
   - otherwise show user prompt,
8. execute or return tool-result error.

Permission decision type:

```ts
type PermissionDecision =
  | { behavior: "allow"; source: "rule" | "tool" | "hook" | "classifier" }
  | { behavior: "deny"; source: "rule" | "tool" | "hook" | "classifier"; reason: string }
  | {
      behavior: "ask"
      message: string
      suggestedAllowRules?: string[]
      suggestedDenyRules?: string[]
    }
```

Decision precedence:

```text
hard deny > explicit deny rule > explicit allow rule > tool-specific safety > hook > classifier/user ask
```

Concrete policy table:

| Operation | `default` | `acceptEdits` | `plan` | `dontAsk` |
|---|---|---|---|---|
| `Read(project file)` | allow | allow | allow | allow |
| `Grep/Glob(project)` | allow | allow | allow | allow |
| `Edit(project file after Read)` | ask | allow | deny | deny |
| `Write(new project file)` | ask | allow | deny | deny |
| `Bash(git status)` | allow if read-only | allow if read-only | allow if read-only | allow if read-only |
| `Bash(npm install)` | ask | ask | deny | deny |
| `Bash(rm -rf *)` | ask or deny by policy | ask or deny by policy | deny | deny |
| `WebFetch(known domain)` | ask or allow by config | ask or allow by config | ask or allow by config | deny if ask |
| Unknown MCP side effect | ask | ask | deny | deny |

## 6.3 Rule Format

Use structured permission rules:

```text
Bash(git status)
Bash(npm test:*)
-Bash(rm *)
+Read(/project/**)
-Write(/etc/**)
```

For MCP:

```text
mcp__server
mcp__server__tool
mcp__server__*
```

## 6.4 Bash Safety Defaults

Recommended concrete settings:

| Setting | Value |
|---|---:|
| Default Bash timeout | `120_000 ms` |
| Max Bash timeout | `600_000 ms` |
| Show progress after | `2_000 ms` |
| Auto-background in assistant mode | `15_000 ms` |
| Block standalone `sleep N` | `N >= 2 seconds` |
| Max subcommands for security check | `50` |
| Max suggested rules for compound command | `5` |

Shell command validation should:

- strip safe wrappers such as `timeout`, `nice`, `nohup`, `time` before rule matching,
- detect `cd` and prevent cwd mutation in subagents,
- classify read-only commands separately,
- treat parse failure as ask/deny, not allow,
- sandbox commands when possible.
