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

| 参数名 | 中文含义 | 单位 | 默认值 | 为什么是这个值 | 触发行为 | 调大后果 | 调小后果 |
|---|---|---|---:|---|---|---|---|
| `maxToolCallsPerTurn` | 单轮工具调用上限 | 个 | `20` | 20 个能覆盖一轮批量读取和搜索，同时防止模型一次性发出失控工具列表。 | assistant 一轮请求工具超过 20 个时，拒绝超出的 tool_use 并返回 budget_exceeded。 | 大批量查询更方便，但工具成本、日志量和失败面上升。 | 模型需要更多轮完成批量任务。 |
| `maxAgentSpawnsPerTurn` | 单轮子 Agent 启动上限 | 个 | `3` | 3 个足够并行调查、实现、验证，避免子 Agent 爆炸式消耗。 | 父 Agent 同一轮启动第 4 个子 Agent 时返回 spawn_limit_exceeded。 | 并行能力更强，但成本、文件冲突和摘要合并压力上升。 | 复杂任务拆分能力下降。 |
| `defaultMaxCostUsd` | 默认单任务费用上限 | 美元 | `5` | 5 美元足够一次中等代码任务，能防止模型循环导致意外高账单。 | 任务累计估算费用超过 5 美元时暂停并要求用户确认继续。 | 长任务更少被打断，但费用失控风险上升。 | 成本更可控，但复杂任务更容易中断。 |
| `rateLimitBucketMs` | 限速统计窗口 | 毫秒 | `60_000` | 1 分钟窗口适合按 provider 每分钟限制做本地排队。 | 每个模型/provider 按 60 秒桶统计请求数和 token。 | 限速更平滑但反应更慢。 | 限速更敏感，但短突发更容易被拒绝。 |
| `maxConcurrentModelCalls` | 模型调用并发上限 | 个 | `4` | 4 个并发支持主 Agent、验证和少量子 Agent，同时避免把 provider 打爆。 | 同时运行的模型调用达到 4 个后，新请求进入队列。 | 吞吐更高，但 429 和成本峰值上升。 | 更稳定，但多 Agent 并行能力下降。 |

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

来源：不适用。本章没有默认模型调用；如果实现需要模型参与，必须复用上下文压缩、权限解释或工具错误恢复章节的固定提示词。

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
