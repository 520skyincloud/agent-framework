# 部署模式

## 设计目标

- 定义本地 CLI、桌面端、服务端、队列 worker、离线评估和企业部署的差异。

## 非目标

- 不接管其它专题的职责。
- 不使用隐藏全局状态。
- 不把失败留给调用方猜测。

## 核心规则

- 本地 CLI 可访问当前 workspace。
- 服务端模式默认禁用任意 Bash。
- 队列 worker 必须有 job timeout。
- 离线评估必须使用 fake provider。
- 企业模式强制审计、配额、secret redaction。
- 部署模式必须影响权限默认值。

## 状态机

~~~mermaid
stateDiagram-v2
  state "读取部署模式" as S0
  state "解析存储" as S1
  state "解析权限 profile" as S2
  state "解析凭据来源" as S3
  state "解析任务执行器" as S4
  state "解析网络策略" as S5
  state "运行 self-check" as S6
  state "生成 runtime config" as S7
  state "启动" as S8
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
type DeploymentMode = "local_cli" | "desktop" | "server" | "worker" | "offline_eval" | "enterprise"
type DeploymentRuntimeConfig = { mode: DeploymentMode; storageRoot: string; permissionProfile: string; allowBash: boolean; auditRequired: boolean; networkPolicy: "allow" | "deny" | "allowlist" }
~~~

## 默认值

| 参数名 | 中文含义 | 单位 | 默认值 | 为什么是这个值 | 触发行为 | 调大后果 | 调小后果 |
|---|---|---|---:|---|---|---|---|
| `localStorageRoot` | 本地存储 | 文本/策略 | `~/.agent` | 该值是第一版实现的固定边界，用来保证行为可预测、可测试、可回放。 | 对应模块执行到该决策点时读取。 | 边界更宽松，但成本、延迟或风险会上升。 | 边界更保守，但更容易提前截断、阻塞或要求确认。 |
| `serverJobTimeoutMs` | 服务端任务 30 分钟 | 毫秒 | `1_800_000` | 这个时间值用于区分正常等待和疑似卡死，第一版优先保证任务不会无限挂起。 | 运行时间或等待时间超过该值时触发超时、刷新、心跳或清理。 | 更不容易误杀慢任务，但卡住时等待更久。 | 故障暴露更快，但慢任务更容易被误判。 |
| `workerHeartbeatMs` | worker 心跳 | 毫秒 | `5_000` | 这个时间值用于区分正常等待和疑似卡死，第一版优先保证任务不会无限挂起。 | 运行时间或等待时间超过该值时触发超时、刷新、心跳或清理。 | 更不容易误杀慢任务，但卡住时等待更久。 | 故障暴露更快，但慢任务更容易被误判。 |
| `enterpriseAuditRequired` | 企业审计必开 | 文本/策略 | `true` | 这个策略值用于给副作用、安全或部署行为一个确定默认，避免实现时出现隐式放行。 | 权限、部署或安全检查进入对应分支时使用。 | 如果改得更宽松，操作更顺滑但安全风险更高。 | 如果改得更保守，安全性更强但需要更多确认。 |
| `serverBashDefault` | 服务端 Bash 默认拒绝 | 文本/策略 | `deny` | 这个策略值用于给副作用、安全或部署行为一个确定默认，避免实现时出现隐式放行。 | 权限、部署或安全检查进入对应分支时使用。 | 如果改得更宽松，操作更顺滑但安全风险更高。 | 如果改得更保守，安全性更强但需要更多确认。 |

## 详细流程

1. 读取 mode。
2. 解析 storage root。
3. 解析 permission profile。
4. 解析 provider credential 来源。
5. 解析 task executor。
6. 解析 network policy。
7. 生成 DeploymentRuntimeConfig。
8. 启动前运行 self-check。

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| mode_unknown | 拒绝启动。 |
| storage_unwritable | 拒绝启动。 |
| missing_credentials | 禁用对应 provider。 |
| server_bash_requested | 拒绝执行。 |
| worker_heartbeat_lost | job 标记 unknown，可重试。 |

## 提示词模板

本章没有默认模型调用；如果实现需要模型参与，必须复用上下文压缩、权限解释或工具错误恢复章节的固定提示词。

## 可实现伪代码

~~~ts
function resolveDeploymentMode(env: Env): DeploymentRuntimeConfig {
  const mode = parseMode(env.AGENT_MODE ?? "local_cli")
  if (mode === "server") return { mode, storageRoot: env.AGENT_STORAGE, permissionProfile: "strict", allowBash: false, auditRequired: true, networkPolicy: "allowlist" }
  if (mode === "offline_eval") return { mode, storageRoot: tempDir(), permissionProfile: "replay", allowBash: false, auditRequired: false, networkPolicy: "deny" }
  if (mode === "enterprise") return { mode, storageRoot: env.AGENT_STORAGE, permissionProfile: "enterprise", allowBash: false, auditRequired: true, networkPolicy: "allowlist" }
  return { mode, storageRoot: "~/.agent", permissionProfile: "ask", allowBash: true, auditRequired: false, networkPolicy: "allow" }
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 本地 CLI | AGENT_MODE 未设置 | allowBash=true，permissionProfile=ask。 |
| 服务端 | AGENT_MODE=server | allowBash=false，networkPolicy=allowlist。 |
| 离线评估 | AGENT_MODE=offline_eval | fake provider，networkPolicy=deny。 |
| 企业 | AGENT_MODE=enterprise | auditRequired=true，secret redaction 必开。 |
| 存储不可写 | storageRoot 无写权限 | storage_unwritable，拒绝启动。 |

## 验收标准

- 有具体默认值。
- 有结构化错误码。
- 有可执行伪代码。
- 测试覆盖正常路径、失败路径和边界值。
