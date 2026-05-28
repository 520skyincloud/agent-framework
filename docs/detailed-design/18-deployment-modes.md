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
| `localStorageRoot` | 本地模式存储根目录 | 路径 | `~/.agent` | 本地状态集中放在用户目录，避免写进项目源码。 | local 模式启动时在该目录创建 sessions、cache、logs。 | 改到项目内便于打包，但容易污染仓库。 | 改到临时目录会削弱恢复和审计。 |
| `serverJobTimeoutMs` | 服务端单任务超时 | 毫秒 | `1_800_000` | 服务端 worker 资源有限，30 分钟后应释放并让任务可恢复。 | server job 超过 30 分钟时停止 worker，写入 timeout 状态。 | 长任务更可能完成，但 worker 被占用更久。 | 服务端资源释放更快，但长任务更容易失败。 |
| `workerHeartbeatMs` | worker 心跳间隔 | 毫秒 | `5_000` | 5 秒心跳能较快发现 worker 崩溃，又不会产生太多写入。 | worker 每 5 秒更新 heartbeat；调度器据此判断是否失联。 | 心跳更少，存储更轻但失联发现更慢。 | 失联发现更快，但心跳写入更多。 |
| `enterpriseAuditRequired` | 企业模式是否强制审计 | 布尔 | `true` | 企业部署需要追踪权限、工具、模型和文件副作用，默认不能关闭。 | enterprise 模式启动时未配置 audit sink 则拒绝启动。 | 若关闭，部署更简单但合规和追责能力下降。 | 保持开启需要配置审计存储。 |
| `serverBashDefault` | 服务端 Bash 默认策略 | 策略 | `deny` | 远程 Bash 影响服务器和租户隔离，默认拒绝最安全。 | server 模式收到 Bash tool_use 时默认 deny，除非显式 allow policy。 | 改成 ask/allow 可支持远程构建，但安全边界复杂度上升。 | 保持 deny 会限制服务端自动化能力。 |

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
