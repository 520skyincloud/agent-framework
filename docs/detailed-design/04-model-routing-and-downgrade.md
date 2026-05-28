# 模型路由与降级

## 设计目标

模型路由负责选择主模型、fallback 模型、compact 模型、classifier 模型、verification 模型，并把每个模型的上下文窗口和输出上限接入预算系统。

## 非目标

- 不实现具体 provider SDK。
- 不让业务代码绕过统一路由。
- 不在降级时保留完整历史。

## 核心规则

- 主模型负责正常推理和工具决策。
- fallback 模型在主模型不可用或 529 连续 3 次后使用。
- compact 模型负责上下文压缩。
- classifier 模型负责低成本分类和权限辅助判断。
- verification 模型负责复核、测试总结、结果审查。
- 默认 capped output 是 8_000。
- escalate retry 是 64_000。
- max output tokens 最多恢复 3 轮。
- prompt too long 的 reactive compact 只 retry 一次。
- compact 已失败 3 次时停止重试。

## 状态机

~~~mermaid
stateDiagram-v2
  [*] --> Resolve
  Resolve --> BudgetCheck
  BudgetCheck --> Main
  Main --> Success
  Main --> EscalateOutput: 输出截断
  EscalateOutput --> Main: 少于 3 次
  Main --> ReactiveCompact: prompt too long
  ReactiveCompact --> Main: 第一次
  Main --> Fallback: 529 连续 3 次
  Main --> Failed: 401 或重试耗尽
  Fallback --> Success
  Fallback --> Failed
~~~

## 数据结构

~~~ts
type ModelRole = "main" | "fallback" | "compact" | "classifier" | "verification"
type ModelSpec = { id: string; role: ModelRole; provider: string; contextWindow: number; maxOutputTokens: number; supportsTools: boolean; supportsStreaming: boolean }
type RetryDecision = { action: "retry_same" | "retry_with_more_output" | "compact_then_retry" | "fallback" | "fail"; delayMs?: number; maxOutputTokens?: number; reason: string }
~~~

## 默认值

| 配置 | 默认值 | 说明 |
|---|---:|---|
| defaultMaxOutputTokens | 8_000 | 普通输出上限。 |
| escalatedMaxOutputTokens | 64_000 | 输出截断恢复上限。 |
| maxOutputRecoveryRounds | 3 | 最多恢复 3 轮。 |
| streamIdleTimeoutMs | 90_000 | 流式空闲超时。 |
| defaultRetries | 10 | 可恢复错误默认重试次数。 |
| baseDelayMs | 500 | 退避起点。 |
| maxNormalDelayMs | 32_000 | 普通重试最大延迟。 |
| fallbackAfterConsecutive529 | 3 | 连续 529 三次后 fallback 或失败。 |

## 详细流程

1. 根据任务类型选择模型角色。
2. 把目标模型 contextWindow 交给上下文预算系统。
3. 从大模型切小模型时，先按目标窗口判断是否 downgrade compact。
4. 默认使用 8_000 输出上限。
5. 输出截断时提升到 64_000，最多 3 轮。
6. prompt too long 时 reactive compact，一次后仍失败就停止。
7. 429、529、5xx、stream idle 按决策表重试。
8. 401 立即失败。
9. 529 连续 3 次触发 fallback；没有 fallback 就明确失败。

| 错误 | 行为 |
|---|---|
| 401 | 立即失败，不重试。 |
| 429 | 指数退避，最多 10 次。 |
| 529 | 指数退避，连续 3 次 fallback。 |
| 5xx | 指数退避，超过上限后 fallback 或失败。 |
| prompt too long | compact 后只重试一次。 |
| stream idle | 90_000 ms 后断开并重试。 |
| max output | 8_000 提升到 64_000，最多 3 轮。 |

## 失败处理

| 失败 | 处理 |
|---|---|
| fallback 上下文更小 | 先 downgrade compact。 |
| compact 模型不可用 | block，要求换模型或清理历史。 |
| 429 超过 10 次 | 返回 rate_limit_exceeded。 |
| 401 | 返回 auth_failed。 |
| stream idle 重复出现 | 计入 5xx 类重试预算。 |

## 提示词模板

~~~text
当前会话需要切换到上下文更小的模型。你只能根据压缩摘要、最新用户指令和可重新读取的文件继续任务。不要假装看过完整历史。如缺少关键信息，先调用工具读取。明确说明历史已被压缩。
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
  if ([429, 529, 500, 502, 503, 504].includes(error.status) && state.retries < 10) return { action: "retry_same", delayMs: Math.min(32_000, 500 * 2 ** state.retries), reason: "transient" }
  return { action: "fail", reason: "retry_exhausted" }
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 输出截断 | outputRecoveryRounds=0 | retry_with_more_output 到 64_000。 |
| 输出截断 3 次 | outputRecoveryRounds=3 | fail。 |
| prompt too long 首次 | reactiveCompactTried=false | compact_then_retry。 |
| prompt too long 二次 | reactiveCompactTried=true | fail。 |
| 529 三次 | consecutive529=3 | fallback。 |
| 401 | status=401（认证失败） | 失败。 |

## 验收标准

- 每个模型角色职责清楚。
- 不同 contextWindow 会进入预算系统。
- 429、529、5xx、401、stream idle 都有决策。
- stream idle timeout 是 90_000 ms。
- default retries 是 10，base delay 是 500 ms，max normal delay 是 32_000 ms。
