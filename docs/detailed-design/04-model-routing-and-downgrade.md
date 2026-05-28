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

| 配置 | 默认值 | 说明 |
|---|---:|---|
| defaultMaxOutputTokens | 8_000 | 普通调用输出上限。 |
| escalatedMaxOutputTokens | 64_000 | 输出截断恢复上限。 |
| maxOutputRecoveryRounds | 3 | 最多恢复 3 轮。 |
| streamIdleTimeoutMs | 90_000 | 流式空闲超时。 |
| defaultRetries | 10 | 默认重试次数。 |
| baseDelayMs | 500 | 退避起点。 |
| maxNormalDelayMs | 32_000 | 最大普通退避。 |
| fallbackAfterConsecutive529 | 3 | 三次 529 后 fallback。 |

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
