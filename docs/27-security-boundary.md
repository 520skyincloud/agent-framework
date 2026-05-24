> 中文标题：安全边界
>
> English title: Security Boundary
>
> 中文导读：本章区分权限和安全，说明路径归一化、workspace 边界、命令解析、sandbox、网络、secret 和审计日志。
>
> English guide: This chapter separates permissions from security and covers path normalization, workspace boundaries, command parsing, sandboxing, network policy, secrets, and audit logs.

# 27. Security Boundary

Permissions are not the same as security. Permissions decide whether to attempt an action; sandboxing limits damage if the decision is wrong.

## 27.1 Security Layers

| Layer | Requirement |
|---|---|
| Path normalization | Resolve symlinks before allow/deny decisions |
| Workspace boundary | Writes outside workspace require explicit permission |
| Read boundary | Secrets and system files require stricter rules |
| Command parsing | Validate shell AST or conservative fallback |
| Sandbox | Use OS/container sandbox for shell where possible |
| Network | Domain allow/deny and redirect policy |
| Secrets | Redact from logs, prompts, analytics |
| Audit | Append permission decisions to audit log |

## 27.2 Secret Handling

Redaction defaults:

| Pattern | Action |
|---|---|
| API keys / bearer tokens | redact in logs and analytics |
| `.env` values | do not send full file unless user explicitly asks |
| SSH private keys | deny read by default |
| cloud credentials | deny read by default |
| cookies/session files | ask with high-risk warning |

Audit record:

```ts
type PermissionAuditRecord = {
  version: 1
  sessionId: string
  toolUseId: string
  toolName: string
  decision: "allow" | "deny" | "ask"
  source: "rule" | "user" | "tool" | "hook" | "classifier"
  rule?: string
  inputSummary: string
  cwd: string
  createdAt: string
}
```

Security test cases:

| Case | Expected |
|---|---|
| write through symlink outside workspace | deny/ask after realpath |
| `timeout 5 rm -rf /tmp/x` | wrapper stripped before rule matching |
| `cd / && rm file` | detect cwd mutation and destructive command |
| read `~/.ssh/id_rsa` | deny by default |
| unsafe web redirect to different port | block |
