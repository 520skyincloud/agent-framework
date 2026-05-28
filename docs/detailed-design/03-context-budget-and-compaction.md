# 上下文预算与压缩

## 设计目标

上下文预算系统负责在每次模型调用前判断：继续、自动压缩、完整压缩、降级压缩或阻塞。它必须用确定数字做决策，不能等 provider 报错后才临时处理。

## 非目标

- 不负责长期记忆排序。
- 不负责业务模型选择。
- 不在小上下文模型里保留完整历史。

## 核心规则

- 默认上下文：200_000 tokens。
- 大上下文：1_000_000 tokens。
- compact 输出 reserve：20_000 tokens。
- 自动压缩缓冲（autoCompactBuffer）：13_000 tokens。
- 手动/阻塞保留量（blockingReserve）：3_000 tokens。
- 警告/错误缓冲（warningBuffer/errorBuffer）：20_000 tokens。
- 自动压缩失败熔断：3 次。
- 有效窗口：effectiveWindow = contextWindow - min(modelMaxOutputTokens, 20_000)。
- 自动压缩阈值：autoCompactAt = effectiveWindow - 13_000。
- 阻塞阈值：blockAt = effectiveWindow - 3_000。
- promptTokens 大于 blockAt 时禁止直接调用模型。

## 状态机

~~~mermaid
stateDiagram-v2
  [*] --> Estimate
  Estimate --> Continue: 小于等于 autoCompactAt
  Estimate --> AutoCompact: 超过 autoCompactAt
  Estimate --> FullCompact: 超过 blockAt
  Estimate --> DowngradeCompact: 切小模型
  AutoCompact --> Continue
  FullCompact --> Continue
  DowngradeCompact --> AggressiveCompact: 第 1 次仍超限
  AggressiveCompact --> TaskStateOnly: 第 2 次仍超限
  TaskStateOnly --> Block: 第 3 次仍超限
  Block --> [*]
~~~

## 数据结构

~~~ts
type ContextBudgetConfig = {
  contextWindow: number
  modelMaxOutputTokens: number
  compactOutputReserve: number
  autoCompactBuffer: number
  blockingReserve: number
  warningBuffer: number
  maxCompactFailures: number
}

type ContextBudgetDecision =
  | { action: "continue"; promptTokens: number; autoCompactAt: number; blockAt: number }
  | { action: "auto_compact"; promptTokens: number }
  | { action: "full_compact"; promptTokens: number }
  | { action: "downgrade_compact"; promptTokens: number; targetWindow: number }
  | { action: "block"; promptTokens: number; reason: string }
~~~

## 默认值

| 场景 | contextWindow | reserve | effectiveWindow | autoCompactAt | blockAt |
|---|---:|---:|---:|---:|---:|
| 200k 模型 | 200_000 | 20_000 | 180_000 | 167_000 | 177_000 |
| 1M 模型 | 1_000_000 | 20_000 | 980_000 | 967_000 | 977_000 |
| 20k 模型 | 20_000 | min(modelMaxOutputTokens, 4_000) | 16_000 | 3_000 | 13_000 |

## 详细流程

1. 每次模型调用前估算 promptTokens。
2. 读取目标模型 contextWindow，不使用历史模型窗口。
3. 200k 模型下：小于等于 167_000 继续；167_001 到 177_000 自动压缩；大于 177_000 完整压缩。
4. 1M 模型下：小于等于 967_000 继续；967_001 到 977_000 自动压缩；大于 977_000 完整压缩。
5. 从 1M 切到 200k：如果当前 prompt 小于等于 167_000，允许继续；167_001 到 177_000 先 compact；大于 177_000 禁止直接调用模型，必须 full compact。
6. 从 1M 或 200k 切到 20k：目标上下文按 20_000 计算，reserve 改为 min(modelMaxOutputTokens, 4_000)，必须进入 downgrade compact。
7. downgrade compact 只保留目标、当前状态、关键文件、未完成任务、工具结果摘要、风险。
8. compact 后仍超限：第 1 次更激进 compact；第 2 次只保留 task state + latest user intent；第 3 次阻塞，要求用户确认清理历史或换回大上下文模型。

## 失败处理

| 失败 | 处理 |
|---|---|
| token 估算器不可用 | 用字符数近似并标记 estimate=true。 |
| 自动 compact 失败 1 次 | 降低输出预算后重试。 |
| compact 连续失败 3 次 | 熔断并 block。 |
| prompt too long | reactive compact 只 retry 一次。 |
| 降级压缩仍超限 | 按三阶段收缩，第三次阻塞。 |

## 提示词模板

### 完整压缩提示词（full compact prompt）

~~~text
你是上下文压缩器。请把会话历史压缩成后续 Agent 可以继续工作的状态摘要。
必须保留：用户最终目标、最近明确指令、已完成步骤、未完成任务、关键文件路径、关键工具结果、权限限制、预算限制、失败风险、不能重复执行的副作用。
必须删除：问候、重复解释、过时计划、大段原始日志、与当前任务无关的历史。
输出格式：用户目标、当前状态、已完成、未完成、关键文件、关键工具结果、风险与约束、下一步建议。
长度上限：20_000 tokens。不要编造。
~~~

### 降级压缩提示词（downgrade compact prompt）

~~~text
你是降级压缩器。目标模型上下文很小，不能保留完整历史。
只保留：最新用户意图、当前任务状态、关键文件、未完成任务、工具结果摘要、风险和不能重复执行的副作用。
删除：完整聊天历史、旧计划、无关工具输出、可重新读取的长文件内容。
目标：能放进 20_000 tokens 模型。
~~~

### 工具结果微压缩提示词（tool-result microcompact prompt）

~~~text
你是工具结果微压缩器。请把工具输出压缩成后续决策需要的最小事实集。
保留：路径、错误码、失败原因、测试结果、关键 diff、风险。
删除：重复日志、安装进度、无关警告。
输出：结论、关键证据、后续影响、原始输出位置。
~~~

### 多 Agent 总结提示词（multi-agent summary prompt）

~~~text
你是子 Agent 结果总结器。请总结子 Agent 的任务、读过或改过的文件、发现的问题、已验证事实、未完成事项和建议下一步。禁止隐藏失败，禁止把猜测写成事实。
~~~

### 压缩失败恢复提示词（compact failure recovery prompt）

~~~text
上一次压缩失败或压缩后仍超限。请执行更激进压缩：只保留最新用户意图、当前任务状态、下一步必须知道的 5 条以内事实、不能丢失的风险或权限限制。输出必须短于上一次的 50%。
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
| 1M 自动压缩 | 970_000 | auto_compact。 |
| 1M 完整压缩 | 978_000 | full_compact。 |
| 切到 20k | 50_000 | downgrade compact。 |
| 三次失败 | compactFailures=3 | block。 |

## 验收标准

- 文档和测试都包含 200_000、1_000_000、20_000、13_000、3_000、167_000、177_000、967_000、977_000。
- 包含 downgrade compact、full compact prompt、tool-result microcompact prompt、multi-agent summary prompt、compact failure recovery prompt。
- blockAt 以上绝不调用模型。
