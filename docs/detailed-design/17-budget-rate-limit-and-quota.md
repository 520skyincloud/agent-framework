# 预算、速率限制与配额

## 设计目标

- 控制 token、费用、工具次数、子 Agent 数、并发和 provider 限速。

## 非目标

- 不接管其它专题的职责。
- 不使用隐藏全局状态。
- 不把失败留给调用方猜测。

## 核心规则

- 每次模型调用前扣预计预算，完成后按实际修正。
- 每 turn 最多 3 个子 Agent。
- 单 turn 工具调用默认最多 20 个。
- 超过预算返回 ask_user 或 abort。
- 429 必须进入 rate limiter。
- 预算事件必须写 observability。

## 状态机

~~~mermaid
stateDiagram-v2
  state "读取预算" as S0
  state "估算消耗" as S1
  state "检查工具次数" as S2
  state "检查子 Agent 数" as S3
  state "检查费用" as S4
  state "检查 provider 限速" as S5
  state "预留预算" as S6
  state "实际用量回填" as S7
  state "发 usage 事件" as S8
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
type BudgetState = { maxCostUsd: number; spentUsd: number; toolCallsThisTurn: number; agentSpawnsThisTurn: number; modelCallsInFlight: number }
~~~

## 默认值

| 配置 | 默认值 | 说明 |
|---|---:|---|
| maxToolCallsPerTurn | 20 | 单轮工具调用上限。 |
| maxAgentSpawnsPerTurn | 3 | 单轮子 Agent 上限。 |
| defaultMaxCostUsd | 5 | 默认单任务费用上限。 |
| rateLimitBucketMs | 60_000 | 限速窗口。 |
| maxConcurrentModelCalls | 4 | 模型并发上限。 |

## 详细流程

1. 读取 session budget。
2. 估算 prompt、output、tool、agent 成本。
3. 如果超过 hard limit，返回 abort。
4. 如果超过 soft limit，返回 ask_user。
5. 通过后 reserve 预算。
6. 调用结束后 reconcile 实际 usage。
7. 收到 429 时更新 provider bucket。

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| cost_exceeded_hard | 中止执行。 |
| cost_exceeded_soft | 请求用户确认。 |
| tool_quota_exceeded | 拒绝新工具调用。 |
| agent_spawn_exceeded | 拒绝新子 Agent。 |
| provider_rate_limited | 等待 retry-after 或排队。 |

## 提示词模板

本章没有默认模型调用；如果实现需要模型参与，必须复用上下文压缩、权限解释或工具错误恢复章节的固定提示词。

## 可实现伪代码

~~~ts
function checkBudget(req: BudgetRequest, state: BudgetState): BudgetDecision {
  if (state.toolCallsThisTurn + req.toolCalls > 20) return { action: "deny", code: "tool_quota_exceeded" }
  if (state.agentSpawnsThisTurn + req.agentSpawns > 3) return { action: "deny", code: "agent_spawn_exceeded" }
  if (state.spentUsd + req.estimatedUsd > state.maxCostUsd) return { action: "ask_user", code: "cost_exceeded_soft" }
  if (state.modelCallsInFlight >= 4) return { action: "queue", code: "model_concurrency_full" }
  return { action: "allow", reservationId: reserve(req) }
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 工具超限 | 本 turn 已 20 个工具，再请求 1 个 | tool_quota_exceeded。 |
| 子 Agent 超限 | 已启动 3 个，再启动 1 个 | agent_spawn_exceeded。 |
| 费用软超限 | spent+estimated > maxCostUsd | ask_user。 |
| 模型并发满 | modelCallsInFlight=4 | queue。 |
| 429 retry-after | provider 返回 retry-after=12 秒 | bucket 延迟至少 12 秒。 |

## 验收标准

- 有具体默认值。
- 有结构化错误码。
- 有可执行伪代码。
- 测试覆盖正常路径、失败路径和边界值。
