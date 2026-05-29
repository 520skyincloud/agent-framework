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

| 参数名 | 中文含义 | 单位 | 默认值 | 为什么是这个值 | 触发行为 | 调大后果 | 调小后果 |
|---|---|---|---:|---|---|---|---|
| `systemPromptReserve` | 系统提示词预算 | tokens | `4_000` | 系统提示词包含身份、权限、输出规则和安全边界，4k 足够表达硬规则。 | prompt assembly 先扣除该预算，再放入可裁剪内容。 | 可写更多规则，但会压缩工具 schema 和最近消息。 | 系统规则可能被删短，模型行为不稳定。 |
| `toolSchemaReserve` | 工具 schema 默认预算 | tokens | `30_000` | 工具 schema 往往比普通文本更长，30k 能容纳核心工具和部分动态工具。 | 注入工具定义时超过预算就按优先级裁剪 optional schema。 | 可暴露更多工具，但会挤掉会话历史。 | 动态工具更早被裁剪，模型可能不知道可用能力。 |
| `memoryReserve` | 记忆预算 | tokens | `8_000` | 8k 能放入少量高相关长期记忆，不让旧偏好压过当前用户意图。 | 记忆检索结果按相关度填充到 8k 后停止。 | 记忆更丰富，但过期信息更容易干扰。 | 个性化和历史偏好更容易丢失。 |
| `projectInstructionReserve` | 项目指令预算 | tokens | `12_000` | 仓库说明、开发规范和局部指令通常比系统提示更长，12k 能覆盖多级文件。 | 拼接项目指令时超过 12k 就按路径优先级裁剪。 | 项目规则更完整，但最近消息空间减少。 | 深层目录规则更容易被忽略。 |
| `recentMessageMinReserve` | 最近消息最低保留预算 | tokens | `40_000` | 最近用户意图和刚产生的工具结果最影响下一步，必须优先保留。 | 裁剪历史时保证最近消息至少保留 40k，除非上下文已必须 compact。 | 最近历史更完整，但老摘要和记忆更早被裁剪。 | 模型可能忘记刚发生的工具结果或用户修正。 |
| `maxOptionalToolSchemas` | 可选工具 schema 最多注入数量 | 个 | `32` | 可选工具太多会让模型选择困难并浪费 schema token，32 是可浏览的上限。 | 动态工具超过 32 个时按相关度只注入前 32 个。 | 工具可见面更大，但 prompt 更长、误选工具概率更高。 | 有用但低频的工具可能不可见。 |

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

来源：不适用。本章没有默认模型调用；如果实现需要模型参与，必须复用上下文压缩、权限解释或工具错误恢复章节的固定提示词。

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
