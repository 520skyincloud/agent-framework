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
| `defaultMaxOutputTokens` | 普通调用输出上限 | tokens | `8_000` | 普通 Agent 回合通常输出计划、代码说明或工具调用，8k 足够且不会过度压缩输入空间。 | 未指定输出上限时发给 provider。 | 长回答更少截断，但上下文预算会少 8k 以上。 | 模型更容易因输出不够而截断。 |
| `escalatedMaxOutputTokens` | 输出截断恢复上限 | tokens | `64_000` | 64k 只用于 max output 恢复，给长总结或大 patch 说明足够空间。 | finish reason 是 max output 时，下一次重试把 max output 提升到该值。 | 截断恢复更强，但费用和等待显著增加。 | 长输出可能恢复失败。 |
| `maxOutputRecoveryRounds` | 输出截断最多恢复轮数 | 次 | `3` | 3 轮覆盖“8k 失败、64k 失败、再次确认失败”的恢复路径，避免无限重试。 | 输出截断恢复次数达到 3 后返回 fail。 | 更可能拿到完整长结果，但可能重复花费。 | 长任务更快失败，需要用户拆分请求。 |
| `streamIdleTimeoutMs` | 流式响应空闲超时 | 毫秒 | `90_000` | 90 秒允许模型长思考和慢流式输出，也能识别断流。 | stream 90 秒无任何事件时中断请求，记录 stream_idle，并进入重试决策表。 | 慢响应更少误杀，但断流发现更晚。 | 长回答更容易被误判 idle。 |
| `defaultRetries` | 可恢复 API 错误默认重试次数 | 次 | `10` | 10 次能覆盖短暂 429、529、5xx 和网络抖动，同时防止无限请求。 | 429、529、5xx 或 stream idle 进入重试表；达到 10 次后 fallback 或 fail。 | 临时故障恢复概率更高，但用户等待和费用上升。 | provider 短抖动更容易直接失败。 |
| `baseDelayMs` | 重试退避起点 | 毫秒 | `500` | 500ms 对瞬时错误恢复足够快，又不会立即打满 provider。 | 第一次可恢复错误后等待 500ms，后续按指数退避增长。 | 请求更温和，但恢复变慢。 | 恢复更快，但可能加重限流。 |
| `maxNormalDelayMs` | 普通指数退避最大等待 | 毫秒 | `32_000` | 32 秒让交互任务还能接受，同时避免 5xx/429 时快速重试。 | 指数退避算出的等待超过 32k 时封顶。 | provider 压力更小，但恢复等待更久。 | 重试更快但更容易继续触发限流。 |
| `fallbackAfterConsecutive529` | 连续 529 触发 fallback 次数 | 次 | `3` | 连续 3 次 overloaded 基本说明主模型当前不可用，应切 fallback 或明确失败。 | `consecutive529 >= 3` 时返回 fallback 决策。 | 主模型恢复机会更多，但会浪费交互时间。 | 更快切 fallback，但可能错过短暂恢复。 |

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
