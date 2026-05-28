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

| 配置 | 默认值 | 说明 |
|---|---:|---|
| localStorageRoot | ~/.agent | 本地存储。 |
| serverJobTimeoutMs | 1_800_000 | 服务端任务 30 分钟。 |
| workerHeartbeatMs | 5_000 | worker 心跳。 |
| enterpriseAuditRequired | true | 企业审计必开。 |
| serverBashDefault | deny | 服务端 Bash 默认拒绝。 |

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
