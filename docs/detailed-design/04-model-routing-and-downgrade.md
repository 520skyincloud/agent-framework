# 模型路由与降级

## 设计目标

- 定义 main、fallback、compact、classifier、verification 五类模型职责。
- 把模型上下文窗口接入预算系统。
- 定义 429、529、5xx、401、stream idle、max output 的恢复策略。

## 非目标

- 不实现 provider SDK。
- 不让业务代码绕过路由。
- 不在降级时保留完整历史。

## 核心规则

- 默认 capped output 为 8_000。
- 输出截断 escalate retry 到 64_000。
- max output tokens 最多恢复 3 轮。
- prompt too long 的 reactive compact 只 retry 一次。
- compact 已失败 3 次停止重试。
- stream idle timeout 为 90_000 ms。
- default retries 为 10。
- base delay 为 500 ms。
- max normal delay 为 32_000 ms。
- 529 连续 3 次触发 fallback 或明确失败。

## 状态机

~~~mermaid
stateDiagram-v2
  state "解析路由" as S0
  state "预算检查" as S1
  state "调用主模型" as S2
  state "输出恢复" as S3
  state "触发式压缩" as S4
  state "fallback" as S5
  state "成功" as S6
  state "失败" as S7
  [*] --> S0
  S0 --> S1
  S1 --> S2
  S2 --> S3
  S3 --> S4
  S4 --> S5
  S5 --> S6
  S6 --> S7
  S7 --> [*]
~~~

## 数据结构

~~~ts
type ModelRole = "main" | "fallback" | "compact" | "classifier" | "verification"
type ModelSpec = { id: string; role: ModelRole; contextWindow: number; maxOutputTokens: number; supportsTools: boolean; supportsStreaming: boolean }
type RetryState = { retries: number; consecutive529: number; outputRecoveryRounds: number; reactiveCompactTried: boolean; compactFailures: number }
type RetryDecision = { action: "retry_same" | "retry_with_more_output" | "compact_then_retry" | "fallback" | "fail"; delayMs?: number; maxOutputTokens?: number; reason: string }
~~~

## 默认值

| 参数名 | 中文含义 | 单位 | 默认值 | 为什么是这个值 | 触发行为 | 调大后果 | 调小后果 |
|---|---|---|---:|---|---|---|---|
| `defaultMaxOutputTokens` | 普通调用输出上限 | tokens | `8_000` | 这个值按 token 预算设置，用来把关键上下文放进模型输入，同时避免某一类内容挤掉最新用户意图。 | 组装 prompt 或计算 token 预算时使用。 | 该类内容可保留更多，但会挤压其它上下文。 | prompt 更紧凑，但可能丢失必要背景。 |
| `escalatedMaxOutputTokens` | 输出截断恢复上限 | tokens | `64_000` | 这个值按 token 预算设置，用来把关键上下文放进模型输入，同时避免某一类内容挤掉最新用户意图。 | 组装 prompt 或计算 token 预算时使用。 | 该类内容可保留更多，但会挤压其它上下文。 | prompt 更紧凑，但可能丢失必要背景。 |
| `maxOutputRecoveryRounds` | 最多恢复 3 轮 | 个/次 | `3` | 这个数量限制用于控制一次任务的执行规模，防止循环、并发或子任务无限扩张。 | 数量达到该值时停止、排队、截断或要求用户确认。 | 吞吐和覆盖面更高，但成本、冲突和排查难度上升。 | 系统更稳，但复杂任务更容易分多轮完成。 |
| `streamIdleTimeoutMs` | 流式响应空闲超时 | 毫秒 | `90_000` | 90 秒允许模型长思考，同时能发现断流。 | 流式输出无事件超过该值时重试。 | 误判断流更少但卡住更久。 | 长响应更容易被误杀。 |
| `defaultRetries` | 默认重试次数 | 次 | `10` | 这个数量限制用于控制一次任务的执行规模，防止循环、并发或子任务无限扩张。 | 数量达到该值时停止、排队、截断或要求用户确认。 | 吞吐和覆盖面更高，但成本、冲突和排查难度上升。 | 系统更稳，但复杂任务更容易分多轮完成。 |
| `baseDelayMs` | 重试退避起点 | 毫秒 | `500` | 500ms 对短暂错误足够快，也不会立即打爆 provider。 | 第一次可恢复错误后等待。 | 更温和但恢复慢。 | 更激进但可能加重限流。 |
| `maxNormalDelayMs` | 最大普通退避 | 毫秒 | `32_000` | 这个时间值用于区分正常等待和疑似卡死，第一版优先保证任务不会无限挂起。 | 运行时间或等待时间超过该值时触发超时、刷新、心跳或清理。 | 更不容易误杀慢任务，但卡住时等待更久。 | 故障暴露更快，但慢任务更容易被误判。 |
| `fallbackAfterConsecutive529` | 三次 529 后 fallback | 次 | `3` | 该值是第一版实现的固定边界，用来保证行为可预测、可测试、可回放。 | 对应模块执行到该决策点时读取。 | 边界更宽松，但成本、延迟或风险会上升。 | 边界更保守，但更容易提前截断、阻塞或要求确认。 |

## 详细流程

1. 按任务类型选 role：normal=main、compact=compact、classify=classifier、verify=verification。
2. 每次调用前把目标模型 contextWindow 交给上下文预算。
3. 从 1M 或 200k 切到 20k 时，先 downgrade compact。
4. 普通调用 maxOutputTokens=8_000。
5. finish reason 为 max output 时提升到 64_000，最多 3 轮。
6. prompt too long 时 reactive compact，只重试一次。
7. 401 立即失败。
8. 429 按 retry-after 或指数退避。
9. 529 连续 3 次 fallback。
10. 5xx 达到 10 次后 fallback 或失败。

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| 401 | auth_failed，不重试。 |
| 429 | rate_limited，最多 10 次。 |
| 529 | provider_overloaded，连续 3 次 fallback。 |
| 5xx | transient_server_error，指数退避。 |
| stream_idle | 90_000 ms 无事件则中断重试。 |
| prompt_too_long | compact_then_retry 一次。 |

## 提示词模板

~~~text
当前会话将切换到上下文更小的模型。你只能根据压缩摘要、最新用户指令和可重新读取的文件继续任务。不要假装看过完整历史；缺信息时先调用工具读取。
~~~

## 可实现伪代码

~~~ts
function decideRetry(error: ModelError, state: RetryState): RetryDecision {
  if (error.status === 401) return { action: "fail", reason: "auth_failed" }
  if (error.code === "prompt_too_long") {
    if (state.reactiveCompactTried || state.compactFailures >= 3) return { action: "fail", reason: "prompt_too_long" }
    return { action: "compact_then_retry", reason: "prompt_too_long" }
  }
  if (error.code === "max_output" && state.outputRecoveryRounds < 3) return { action: "retry_with_more_output", maxOutputTokens: 64_000, reason: "max_output" }
  if (error.status === 529 && state.consecutive529 >= 3) return { action: "fallback", reason: "provider_overloaded" }
  if ([429,529,500,502,503,504].includes(error.status) && state.retries < 10) return { action: "retry_same", delayMs: Math.min(32_000, 500 * 2 ** state.retries), reason: "transient" }
  return { action: "fail", reason: "retry_exhausted" }
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 输出截断首轮 | outputRecoveryRounds=0 | 64_000 重试。 |
| 输出截断三轮 | outputRecoveryRounds=3 | fail。 |
| prompt too long 首次 | reactiveCompactTried=false | compact_then_retry。 |
| 529 三次 | consecutive529=3 | fallback。 |
| 401 | status=401（认证失败） | 失败。 |

## 验收标准

- 五类模型职责明确。
- 所有模型都有 contextWindow。
- 错误决策表覆盖 429、529、5xx、401、stream idle。
- 529 三次不会继续打主模型。
