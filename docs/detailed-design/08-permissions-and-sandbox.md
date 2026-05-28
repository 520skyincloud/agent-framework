# 权限与沙箱

## 设计目标

- 在所有副作用执行前做确定性权限决策，并记录审计。

## 非目标

- 不接管其它专题的职责。
- 不使用隐藏全局状态。
- 不把失败留给调用方猜测。

## 核心规则

- hard deny 优先级最高。
- 显式 deny 高于 allow。
- Edit/Write 必须限制在 workspace 内。
- Bash 危险命令默认 ask 或 deny。
- MCP 工具按 mcp__server__tool 命名空间授权。
- 每次 ask/allow/deny 都写 audit。

## 状态机

~~~mermaid
stateDiagram-v2
  state "接收权限请求" as S0
  state "hard deny 检查" as S1
  state "显式 deny 匹配" as S2
  state "显式 allow 匹配" as S3
  state "工具安全检查" as S4
  state "hook 或 classifier" as S5
  state "用户确认" as S6
  state "写审计" as S7
  state "返回决策" as S8
  [*] --> S0
  S0 --> S1
  S1 --> S2
  S2 --> S3
  S3 --> S4
  S4 --> S5
  S5 --> S6
  S6 --> S7
  S7 --> S8
  S8 --> [*]
~~~

## 数据结构

~~~ts
type PermissionDecision = { decision: "allow" | "deny" | "ask"; reason: string; ruleId?: string; expiresAt?: string }
type PermissionAuditRecord = { sessionId: string; toolUseId: string; toolName: string; decision: string; inputSummary: string; cwd: string; createdAt: string }
~~~

## 默认值

| 配置 | 默认值 | 说明 |
|---|---:|---|
| permissionMode | ask | 默认需要用户确认副作用。 |
| allowTTLSeconds | 3_600 | 用户本次允许的有效期。 |
| auditRetentionDays | 30 | 权限审计保留天数。 |
| dangerousBashAsk | true | 危险 bash 默认询问。 |
| outsideWorkspaceWrite | deny | 工作区外写入默认拒绝。 |

## 详细流程

1. 检查 hard deny：删除根目录、读取 secret、写系统目录。
2. 匹配 deny rules。
3. 匹配 allow rules。
4. 调用工具专属 safety。
5. 调用 hook 或 classifier。
6. 如果仍不确定，返回 ask。
7. 写 PermissionAuditRecord。

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| secret_detected | deny，并隐藏具体 secret。 |
| outside_workspace | 拒绝。 |
| ambiguous_bash | 询问用户。 |
| noninteractive_ask | deny，非交互无法询问。 |
| audit_write_failed | 允许结果不生效，返回 permission_audit_failed。 |

## 提示词模板

### 权限解释提示词

~~~text
你是权限解释器。请用简短中文解释为什么这个工具调用需要允许、被拒绝或需要用户确认。
必须包含：工具名、目标资源、风险、如果允许会发生什么、如果拒绝会怎样继续。
不要泄露 secret 原文。不要夸大风险。不要替用户做授权决定。
~~~

## 可实现伪代码

~~~ts
async function decidePermission(req: PermissionRequest): Promise<PermissionDecision> {
  if (matchesHardDeny(req)) return audit(req, { decision: "deny", reason: "hard_deny" })
  const deny = matchRule(req, req.rules.deny)
  if (deny) return audit(req, { decision: "deny", reason: "explicit_deny", ruleId: deny.id })
  const allow = matchRule(req, req.rules.allow)
  if (allow && toolSafety(req).safe) return audit(req, { decision: "allow", reason: "explicit_allow", ruleId: allow.id })
  const safety = toolSafety(req)
  if (!safety.safe) return audit(req, { decision: safety.askable ? "ask" : "deny", reason: safety.reason })
  return audit(req, { decision: "ask", reason: "no_matching_rule" })
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 硬拒绝 | Bash(rm -rf /) | deny，reason=hard_deny。 |
| 工作区外写 | Write(/etc/hosts) | deny，reason=outside_workspace。 |
| 明确允许 | Read(/project/src/a.ts) 命中 allow | allow，并写 audit。 |
| 非交互 ask | Bash(npm install) 需要询问但 nonInteractive=true | deny，reason=noninteractive_ask。 |
| MCP 授权 | mcp__github__create_issue 未在 allow | ask 或 deny，不直接执行。 |

## 验收标准

- 有具体默认值。
- 有结构化错误码。
- 有可执行伪代码。
- 测试覆盖正常路径、失败路径和边界值。
