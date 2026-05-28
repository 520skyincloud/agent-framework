# Prompt 组装

## 设计目标

- 组装 provider 请求，确保内容顺序、预算裁剪和工具 schema 都确定。
- 把上下文预算结果转成可发送的消息数组。

## 非目标

- 不接管其它专题的职责。
- 不使用隐藏全局状态。
- 不把失败留给调用方猜测。

## 核心规则

- 系统提示词永远第一。
- 必须工具 schema 永远保留；可选工具 schema 可裁剪。
- 最新用户消息永远保留。
- compact summary 放在历史消息之前。
- 每次组装必须输出 tokenBreakdown。
- 超预算时裁剪顺序固定：可选工具、低相关记忆、旧工具结果、旧消息。

## 状态机

~~~mermaid
stateDiagram-v2
  state "接收组装请求" as S0
  state "计算有效窗口" as S1
  state "加入系统提示词" as S2
  state "加入必需工具" as S3
  state "加入压缩摘要" as S4
  state "加入记忆和项目指令" as S5
  state "加入最近消息" as S6
  state "按顺序裁剪" as S7
  state "输出 provider 请求或 compact 决策" as S8
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
type PromptAssemblyResult = { providerMessages: ProviderMessage[]; tokenBreakdown: Record<string, number>; dropped: DroppedContextItem[]; decision: "send" | "compact" | "block" }
~~~

## 默认值

| 配置 | 默认值 | 说明 |
|---|---:|---|
| systemPromptReserve | 4_000 | 系统提示词预算。 |
| toolSchemaReserve | 30_000 | 工具 schema 默认预算。 |
| memoryReserve | 8_000 | 记忆预算。 |
| projectInstructionReserve | 12_000 | 项目指令预算。 |
| recentMessageMinReserve | 40_000 | 最近消息最低保留预算。 |
| maxOptionalToolSchemas | 32 | 可选工具 schema 最多注入数量。 |

## 详细流程

1. 读取模型预算 effectiveWindow。
2. 加入 system prompt。
3. 加入强制工具 schema：Read、Grep、Glob、Bash、Edit、Write、TodoWrite。
4. 加入 latest compact summary。
5. 按相关度加入记忆，最多 8_000 tokens。
6. 加入项目指令，最多 12_000 tokens。
7. 从新到旧加入消息，遇到大工具结果用 external marker。
8. 超预算时按固定顺序裁剪；仍超预算则返回 need_compact。

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| missing_system_prompt | 失败，不能调用模型。 |
| tool_schema_too_large | 裁剪可选工具；必需工具仍超限则 block。 |
| latest_user_message_too_large | 先 microcompact 附件或要求用户拆分。 |
| prompt_still_too_large | 返回 need_compact。 |

## 提示词模板

本章没有默认模型调用；如果实现需要模型参与，必须复用上下文压缩、权限解释或工具错误恢复章节的固定提示词。

## 可实现伪代码

~~~ts
function assemblePrompt(input: PromptAssemblyInput): PromptAssemblyResult {
  const b = new PromptBuilder(input.effectiveWindow)
  b.mustAdd("system", input.systemPrompt, 4_000)
  b.mustAdd("tools.required", requiredToolSchemas(input.tools), 30_000)
  b.tryAdd("compact", input.latestCompactSummary, 20_000)
  b.tryAddRanked("memory", input.memories, 8_000)
  b.tryAdd("project", input.projectInstructions, 12_000)
  b.addRecentMessagesNewestFirst(input.messages, { minReserve: 40_000, externalizeToolResults: true })
  if (b.overBudget()) b.dropInOrder(["tools.optional", "memory.low", "toolResults.old", "messages.old"])
  if (b.overBudget()) return b.result("compact")
  return b.result("send")
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 必需工具保留 | 工具 schema 总量 35_000 tokens，其中必需 28_000 | 保留必需工具，裁剪可选工具。 |
| 最新用户消息过大 | latest user message 60_000 tokens | 返回 latest_user_message_too_large，不丢弃最新用户消息。 |
| 记忆超预算 | 记忆候选 30 条共 20_000 tokens | 按相关度截断到 8_000 tokens。 |
| 仍然超预算 | 裁剪可选工具、低相关记忆、旧工具结果后仍超 effectiveWindow | 返回 decision=compact。 |
| 大工具结果 | 历史中有 80_000 字符 tool_result | 替换成 external marker，tokenBreakdown 记录 toolResults.externalized。 |

## 验收标准

- 有具体默认值。
- 有结构化错误码。
- 有可执行伪代码。
- 测试覆盖正常路径、失败路径和边界值。
