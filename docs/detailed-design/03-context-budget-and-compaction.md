# 上下文预算与压缩

## 设计目标

- 在每次模型调用前判断继续、压缩、降级压缩或阻塞。
- 用确定阈值处理 200k、1M、20k 模型切换。
- 提供可直接使用的压缩提示词。

## 非目标

- 不负责长期记忆排序。
- 不负责业务模型选择。
- 不在 20k 模型里保留完整历史。

## 核心规则

- 默认上下文 200_000 tokens。
- 大上下文 1_000_000 tokens。
- compact 输出 reserve 20_000 tokens。
- 自动压缩缓冲（auto compact buffer）：13_000 tokens。
- 手动/阻塞保留量（manual/blocking reserve）：3_000 tokens。
- 警告/错误缓冲（warning/error buffer）：20_000 tokens。
- 自动压缩失败 3 次熔断。
- 有效窗口公式：effectiveWindow = contextWindow - min(modelMaxOutputTokens, 20_000)。
- 自动压缩阈值公式：autoCompactAt = effectiveWindow - 13_000。
- 阻塞阈值公式：blockAt = effectiveWindow - 3_000。

## 状态机

~~~mermaid
stateDiagram-v2
  state "估算 tokens" as S0
  state "允许继续" as S1
  state "自动压缩" as S2
  state "完整压缩" as S3
  state "降级压缩" as S4
  state "更激进压缩" as S5
  state "只保留任务状态" as S6
  state "阻塞" as S7
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
type ContextBudgetDecision =
  | { action: "continue"; promptTokens: number; autoCompactAt: number; blockAt: number }
  | { action: "auto_compact"; promptTokens: number }
  | { action: "full_compact"; promptTokens: number }
  | { action: "downgrade_compact"; promptTokens: number; targetWindow: number }
  | { action: "block"; promptTokens: number; reason: string }

type CompactSummary = {
  goal: string
  currentState: string
  keyFiles: string[]
  openTasks: string[]
  toolResultSummaries: string[]
  risks: string[]
}
~~~

## 默认值

| 配置 | 默认值 | 说明 |
|---|---:|---|
| 200k 自动压缩阈值（autoCompactAt） | 167_000 | 200_000 - 20_000 - 13_000。 |
| 200k 阻塞阈值（blockAt） | 177_000 | 200_000 - 20_000 - 3_000。 |
| 1M 自动压缩阈值（autoCompactAt） | 967_000 | 1_000_000 - 20_000 - 13_000。 |
| 1M 阻塞阈值（blockAt） | 977_000 | 1_000_000 - 20_000 - 3_000。 |
| 20k.reserve | min(modelMaxOutputTokens, 4_000) | 小模型不使用 20_000 reserve。 |

## 详细流程

1. 200k 下 prompt <= 167_000 继续。
2. 200k 下 167_001 到 177_000 自动压缩。
3. 200k 下 > 177_000 禁止调用模型，执行 full compact。
4. 1M 下 prompt <= 967_000 继续。
5. 1M 下 967_001 到 977_000 自动压缩。
6. 1M 下 > 977_000 执行 full compact。
7. 1M 切 200k 时按 200k 阈值重新判断。
8. 1M 或 200k 切 20k 时必须 downgrade compact，只保留目标、状态、关键文件、未完成任务、工具摘要、风险。
9. 降级压缩仍超限：第 1 次更激进，第 2 次只保留 task state + latest user intent，第 3 次阻塞。

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| token_estimator_unavailable | 用字符近似，并写 estimate=true。 |
| compact_failed_once | 降低输出预算重试。 |
| compact_failed_three_times | block，要求用户确认清理历史或换大模型。 |
| provider_prompt_too_long | reactive compact 只重试一次。 |
| downgrade_still_too_large | 进入三阶段降级压缩。 |

## 提示词模板

### 完整压缩提示词（full compact prompt）

~~~text
你是上下文压缩器。保留用户目标、最近指令、当前状态、已完成、未完成、关键文件、工具结果摘要、风险、不能重复执行的副作用。删除问候、重复解释、过期计划和长日志。输出不超过 20_000 tokens。
~~~

### 降级压缩提示词（downgrade compact prompt）

~~~text
你是降级压缩器。目标模型只有 20_000 tokens。只保留最新用户意图、当前任务状态、关键文件、未完成任务、工具结果摘要和风险。不要保留完整历史。
~~~

### 工具结果微压缩提示词（tool-result microcompact prompt）

~~~text
把工具输出压缩为：结论、关键证据、后续影响、原始输出位置。删除重复日志。
~~~

### 多 Agent 总结提示词（multi-agent summary prompt）

~~~text
总结子 Agent 的任务、文件、发现、验证事实、失败、不确定项和下一步建议。禁止把猜测写成事实。
~~~

### 压缩失败恢复提示词（compact failure recovery prompt）

~~~text
上一次压缩后仍超限。只保留最新用户意图、当前任务状态、5 条以内关键事实和风险。长度必须少于上一次的 50%。
~~~

## 可实现伪代码

~~~ts
function decideContextAction(input: { promptTokens: number; contextWindow: number; modelMaxOutputTokens: number; compactFailures: number; isDowngrade: boolean }): ContextBudgetDecision {
  const reserveCap = input.contextWindow <= 20_000 ? 4_000 : 20_000
  const reserve = Math.min(input.modelMaxOutputTokens, reserveCap)
  const effectiveWindow = input.contextWindow - reserve
  const autoCompactAt = effectiveWindow - 13_000
  const blockAt = effectiveWindow - 3_000
  if (input.compactFailures >= 3) return { action: "block", promptTokens: input.promptTokens, reason: "compact_failed_three_times" }
  if (input.isDowngrade && input.promptTokens > autoCompactAt) return { action: "downgrade_compact", promptTokens: input.promptTokens, targetWindow: input.contextWindow }
  if (input.promptTokens <= autoCompactAt) return { action: "continue", promptTokens: input.promptTokens, autoCompactAt, blockAt }
  if (input.promptTokens <= blockAt) return { action: "auto_compact", promptTokens: input.promptTokens }
  return { action: "full_compact", promptTokens: input.promptTokens }
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 200k 继续 | 167_000 | continue。 |
| 200k 自动压缩 | 170_000 | auto_compact。 |
| 200k 完整压缩 | 178_000 | full_compact。 |
| 1M 继续 | 967_000 | continue。 |
| 1M 完整压缩 | 978_000 | full_compact。 |
| 20k 降级 | 50_000 | downgrade compact。 |

## 验收标准

- 包含 200_000、1_000_000、20_000、13_000、3_000、167_000、177_000、967_000、977_000。
- 包含 downgrade compact 和 full compact prompt。
- blockAt 以上绝不调用模型。
