# 上下文预算与压缩

## 设计目标

- 在每次模型调用前判断继续、warning、自动压缩、完整压缩、降级压缩或阻塞。
- 用确定阈值处理 200k、1M、20k 模型切换。
- 让实现者知道每个数字的含义、单位、触发动作和调参后果。
- 提供可直接使用的压缩提示词和失败恢复分支。

## 非目标

- 不负责长期记忆排序。
- 不负责业务模型选择。
- 不在 20k 模型里保留完整历史。

## 核心规则

- 默认上下文窗口是 `200_000 tokens`。
- 大上下文窗口是 `1_000_000 tokens`。
- 普通大模型的 compact 输出 reserve 是 `20_000 tokens`。
- 20k 小模型的 reserve 改为 `min(modelMaxOutputTokens, 4_000)`。
- 自动压缩缓冲是 `13_000 tokens`。
- 阻塞前最后保留空间是 `3_000 tokens`。
- warning/error 提前量是 `20_000 tokens`。
- 自动压缩连续失败 `3` 次后熔断。
- `promptTokens > blockAt` 时禁止直接调用模型。

## 状态机

~~~mermaid
stateDiagram-v2
  state "估算 prompt tokens" as Estimate
  state "计算有效窗口" as ComputeWindow
  state "安全继续" as Continue
  state "发出 warning" as Warn
  state "自动压缩" as AutoCompact
  state "完整压缩" as FullCompact
  state "降级压缩" as DowngradeCompact
  state "更激进压缩" as AggressiveCompact
  state "只保留任务状态" as TaskStateOnly
  state "阻塞" as Block

  [*] --> Estimate
  Estimate --> ComputeWindow
  ComputeWindow --> Continue: prompt <= warningThreshold
  ComputeWindow --> Warn: warningThreshold < prompt <= autoCompactAt
  ComputeWindow --> AutoCompact: autoCompactAt < prompt <= blockAt
  ComputeWindow --> FullCompact: prompt > blockAt
  ComputeWindow --> DowngradeCompact: 目标模型窗口变小
  DowngradeCompact --> AggressiveCompact: 第 1 次仍超限
  AggressiveCompact --> TaskStateOnly: 第 2 次仍超限
  TaskStateOnly --> Block: 第 3 次仍超限
  Continue --> [*]
  Warn --> [*]
  AutoCompact --> [*]
  FullCompact --> [*]
  Block --> [*]
~~~

## 数据结构

~~~ts
type ContextBudgetInput = {
  promptTokens: number
  contextWindow: number
  modelMaxOutputTokens: number
  compactFailures: number
  isDowngrade: boolean
  targetModel: string
}

type ContextBudgetDecision =
  | { action: "continue"; promptTokens: number; warningThreshold: number; autoCompactAt: number; blockAt: number }
  | { action: "warn"; promptTokens: number; warningThreshold: number }
  | { action: "auto_compact"; promptTokens: number; autoCompactAt: number }
  | { action: "full_compact"; promptTokens: number; blockAt: number }
  | { action: "downgrade_compact"; promptTokens: number; targetWindow: number; targetModel: string }
  | { action: "block"; promptTokens: number; reason: string }

type CompactSummary = {
  goal: string
  latestUserIntent: string
  currentState: string
  keyFiles: string[]
  openTasks: string[]
  toolResultSummaries: string[]
  risks: string[]
  irreversibleSideEffects: string[]
}
~~~

## 默认值

| 参数名 | 中文含义 | 单位 | 默认值 | 为什么是这个值 | 触发行为 | 调大后果 | 调小后果 |
|---|---|---|---:|---|---|---|---|
| `defaultContextWindow` | 默认模型上下文窗口 | tokens | `200_000` | 200k 能覆盖大多数编程任务，又不假设所有 provider 都支持 1M。 | 未显式指定模型窗口时使用。 | 长历史更少压缩，但成本和延迟更高。 | 长任务更容易提前 compact。 |
| `largeContextWindow` | 大上下文模型窗口 | tokens | `1_000_000` | 给长仓库、多 Agent 汇总和超长 transcript 使用。 | 路由到大上下文模型时使用。 | 可保留更多历史，但调用更慢更贵。 | 大会话更早进入压缩。 |
| `compactOutputReserve` | 大模型输出/摘要预留上限 | tokens | `20_000` | 20k 足够输出完整压缩摘要和长回答，同时不会无限侵占 prompt 空间。 | 计算 `effectiveWindow` 时扣除。 | 输出更安全，但 prompt 可用空间减少。 | prompt 空间更大，但输出截断风险上升。 |
| `smallModelReserveCap` | 20k 小模型输出预留上限 | tokens | `4_000` | 小模型总窗口只有 20k，继续预留 20k 会让 prompt 空间归零。 | 目标模型 `contextWindow <= 20_000` 时使用。 | 小模型输出更充分，但可输入内容更少。 | 输出更容易截断，模型可能无法完整说明恢复步骤。 |
| `autoCompactBuffer` | 自动压缩缓冲 | tokens | `13_000` | 给 token 估算误差、工具 schema 和 provider 包装留安全余量。 | prompt 超过 `autoCompactAt` 时触发自动 compact。 | 更早压缩，安全但频繁。 | 更晚压缩，provider 拒绝风险上升。 |
| `blockingReserve` | 阻塞前最后保留空间 | tokens | `3_000` | 给错误解释、恢复消息和 provider 包装留下最后空间。 | prompt 超过 `blockAt` 时禁止直接调用模型。 | block 更早触发，用户更常需要 compact 或换模型。 | block 更晚触发，provider 返回 prompt too long 的风险更高。 |
| `warningBuffer` | warning 提前量 | tokens | `20_000` | 在真正压缩前给 UI 和用户留足操作空间。 | prompt 超过 `warningThreshold` 时发 warning。 | 更早提醒，可能更打扰。 | 更晚提醒，用户更难处理。 |
| `maxCompactFailures` | 自动压缩连续失败熔断次数 | 次 | `3` | 3 次能覆盖偶发失败，同时避免无限循环和成本失控。 | 第 3 次失败后 block。 | 恢复机会更多，但成本更高。 | 更快失败，但偶发问题更容易中断任务。 |

## 公式解释

| 公式 | 变量解释 | 结果含义 | 后续动作 |
|---|---|---|---|
| `reserve = min(modelMaxOutputTokens, 20_000)` | `modelMaxOutputTokens` 是本次输出上限；`20_000` 是大模型预留封顶。 | 模型输出和压缩摘要最多预留多少空间。 | 用来计算有效窗口。 |
| `effectiveWindow = contextWindow - reserve` | `contextWindow` 是目标模型总窗口；`reserve` 是输出预留。 | prompt 真正能安全使用的空间。 | 所有阈值从这里派生。 |
| `autoCompactAt = effectiveWindow - 13_000` | `13_000` 是自动压缩缓冲。 | prompt 超过该值后，应先压缩再调用模型。 | 返回 `auto_compact`。 |
| `blockAt = effectiveWindow - 3_000` | `3_000` 是最后阻塞余量。 | prompt 超过该值后，直接调用模型风险太高。 | 返回 `full_compact` 或 `block`。 |
| `warningThreshold = autoCompactAt - 20_000` | `20_000` 是提前提醒空间。 | prompt 接近自动压缩线。 | 返回 `warn`，但不改写上下文。 |

## 详细流程

1. 估算当前 promptTokens。
2. 读取目标模型的 `contextWindow`，注意使用目标模型，不使用历史模型。
3. 如果目标模型窗口小于等于 `20_000`，reserve 使用 `min(modelMaxOutputTokens, 4_000)`。
4. 否则 reserve 使用 `min(modelMaxOutputTokens, 20_000)`。
5. 计算 `effectiveWindow`、`warningThreshold`、`autoCompactAt`、`blockAt`。
6. 如果 compactFailures 已达到 3，返回 `block`。
7. 如果正在从大模型切到小模型，并且 prompt 超过目标模型 `warningThreshold`，返回 `downgrade_compact`。
8. 如果 prompt 小于等于 warningThreshold，返回 `continue`。
9. 如果 prompt 小于等于 autoCompactAt，返回 `warn`。
10. 如果 prompt 小于等于 blockAt，返回 `auto_compact`。
11. 如果 prompt 大于 blockAt，返回 `full_compact`，禁止直接调用模型。

## 计算例子

### 200k 模型

| 步骤 | 计算 | 结果 | 解释 |
|---|---|---:|---|
| 总窗口 | `contextWindow` | `200_000` | 目标模型窗口。 |
| 输出预留 | `min(modelMaxOutputTokens, 20_000)` | `20_000` | 给模型输出和摘要留空间。 |
| 有效窗口 | `200_000 - 20_000` | `180_000` | prompt 安全可用空间。 |
| 自动压缩 | `180_000 - 13_000` | `167_000` | 超过后自动 compact。 |
| 阻塞限制 | `180_000 - 3_000` | `177_000` | 超过后禁止直接调用模型。 |
| warning | `167_000 - 20_000` | `147_000` | 超过后发 warning。 |

### 1M 模型

| 步骤 | 计算 | 结果 | 解释 |
|---|---|---:|---|
| 总窗口 | `contextWindow` | `1_000_000` | 大上下文窗口。 |
| 输出预留 | `min(modelMaxOutputTokens, 20_000)` | `20_000` | 大模型仍只预留 20k。 |
| 有效窗口 | `1_000_000 - 20_000` | `980_000` | prompt 安全可用空间。 |
| 自动压缩 | `980_000 - 13_000` | `967_000` | 超过后自动 compact。 |
| 阻塞限制 | `980_000 - 3_000` | `977_000` | 超过后 full compact。 |
| warning | `967_000 - 20_000` | `947_000` | 超过后发 warning。 |

### 20k 模型

| 步骤 | 计算 | 结果 | 解释 |
|---|---|---:|---|
| 总窗口 | `contextWindow` | `20_000` | 目标小模型窗口。 |
| 输出预留 | `min(modelMaxOutputTokens, 4_000)` | `4_000` | 小模型不能预留 20k。 |
| 有效窗口 | `20_000 - 4_000` | `16_000` | prompt 安全可用空间。 |
| 自动压缩 | `16_000 - 13_000` | `3_000` | 超过 3k 就应降级压缩。 |
| 阻塞限制 | `16_000 - 3_000` | `13_000` | 超过后不能直接调用。 |

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| `token_estimator_unavailable` | 用字符近似，并写 `estimate=true`；中文按每 1.5 字约 1 token，英文按每 4 字符约 1 token。 |
| `compact_failed_once` | 降低输出预算后重试一次，并记录 compactFailureCount。 |
| `compact_failed_three_times` | 熔断并 block，要求用户确认清理历史或换大模型。 |
| `provider_prompt_too_long` | reactive compact 只重试一次；再次失败就 block。 |
| `downgrade_still_too_large` | 第 1 次更激进 compact；第 2 次只保留 task state + latest user intent；第 3 次 block。 |

## 提示词模板

### 完整压缩提示词（full compact prompt）

~~~text
你是上下文压缩器。请把会话历史压缩成后续 Agent 可以继续工作的状态摘要。
必须保留：用户最终目标、最近明确指令、当前任务状态、已完成步骤、未完成任务、关键文件路径、关键工具结果、权限限制、预算限制、失败风险、不能重复执行的副作用。
必须删除：问候、重复解释、过期计划、大段原始日志、与当前任务无关的历史。
输出结构：用户目标、当前状态、已完成、未完成、关键文件、关键工具结果、风险与约束、下一步建议。
长度上限：20_000 tokens。不要编造。
~~~

### 降级压缩提示词（downgrade compact prompt）

~~~text
你是降级压缩器。目标模型只有 20_000 tokens，不能保留完整历史。
只保留：最新用户意图、当前任务状态、关键文件、未完成任务、影响下一步的工具结果摘要、风险和不能重复执行的副作用。
删除：完整聊天历史、旧计划、无关工具输出、可重新读取的长文件内容。
如果缺少信息，后续 Agent 应通过工具重新读取，不要靠猜。
~~~

### 工具结果微压缩提示词（tool-result microcompact prompt）

~~~text
把工具输出压缩为后续决策需要的最小事实集。
输出：结论、关键证据、后续影响、原始输出位置。
删除：重复日志、安装进度、无关警告、长列表中的低价值项目。
~~~

### 多 Agent 总结提示词（multi-agent summary prompt）

~~~text
总结子 Agent 的任务、使用工具、读过或改过的文件、确定事实、失败、不确定项、风险和建议下一步。
禁止包含完整聊天历史。禁止把猜测写成事实。最多 8_000 tokens。
~~~

### 压缩失败恢复提示词（compact failure recovery prompt）

~~~text
上一次压缩后仍超限。请执行更激进压缩。
只保留：最新用户意图、当前任务状态、下一步必须知道的 5 条以内事实、不能丢失的风险或权限限制。
输出必须少于上一次压缩结果的 50%。
~~~

## 可实现伪代码

~~~ts
function decideContextAction(input: ContextBudgetInput): ContextBudgetDecision {
  const reserveCap = input.contextWindow <= 20_000 ? 4_000 : 20_000
  const reserve = Math.min(input.modelMaxOutputTokens, reserveCap)
  const effectiveWindow = input.contextWindow - reserve
  const autoCompactAt = effectiveWindow - 13_000
  const blockAt = effectiveWindow - 3_000
  const warningThreshold = autoCompactAt - 20_000

  if (input.compactFailures >= 3) {
    return { action: "block", promptTokens: input.promptTokens, reason: "compact_failed_three_times" }
  }
  if (input.isDowngrade && input.promptTokens > warningThreshold) {
    return { action: "downgrade_compact", promptTokens: input.promptTokens, targetWindow: input.contextWindow, targetModel: input.targetModel }
  }
  if (input.promptTokens <= warningThreshold) {
    return { action: "continue", promptTokens: input.promptTokens, warningThreshold, autoCompactAt, blockAt }
  }
  if (input.promptTokens <= autoCompactAt) {
    return { action: "warn", promptTokens: input.promptTokens, warningThreshold }
  }
  if (input.promptTokens <= blockAt) {
    return { action: "auto_compact", promptTokens: input.promptTokens, autoCompactAt }
  }
  return { action: "full_compact", promptTokens: input.promptTokens, blockAt }
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 200k 安全区 | `promptTokens=140_000` | `continue`。 |
| 200k warning | `promptTokens=150_000` | `warn`。 |
| 200k 自动压缩 | `promptTokens=170_000` | `auto_compact`。 |
| 200k 完整压缩 | `promptTokens=178_000` | `full_compact`，不调用模型。 |
| 1M 自动压缩 | `promptTokens=970_000` | `auto_compact`。 |
| 1M 完整压缩 | `promptTokens=978_000` | `full_compact`。 |
| 20k 降级 | 从 200k 会话切到 20k，`promptTokens=50_000` | `downgrade_compact`。 |
| compact 三次失败 | `compactFailures=3` | `block`。 |

## 验收标准

- 文档解释 `effectiveWindow`、`autoCompactAt`、`blockAt`、`warningThreshold` 每个变量的含义。
- 包含 `200_000`、`1_000_000`、`20_000`、`13_000`、`3_000`、`167_000`、`177_000`、`967_000`、`977_000`。
- 包含 200k、1M、20k 三个完整计算例子。
- 包含 downgrade compact、full compact prompt、tool-result microcompact prompt、multi-agent summary prompt、compact failure recovery prompt。
- blockAt 以上绝不调用模型。
