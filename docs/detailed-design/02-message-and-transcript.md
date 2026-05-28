# 消息与 Transcript

## 设计目标

- 定义内部消息格式和 provider 消息格式的转换边界。
- 让 transcript 成为恢复和回放的事实来源。
- 保证 tool_use 与 tool_result 永远可校验。

## 非目标

- 不把 UI 临时状态写入 transcript。
- 不把大型工具输出全文塞进主 transcript。
- 不在消息层决定模型重试。

## 核心规则

- transcript 只能 append-only。
- 每条记录必须有 sequence、messageId、sha256。
- assistant 内多个 tool_use 的结果必须按原顺序追加。
- 工具输出超过 50_000 字符时落盘。
- 恢复时发现损坏，只能追加 repair message，不改旧记录。

## 状态机

~~~mermaid
stateDiagram-v2
  state "追加消息" as S0
  state "计算 hash" as S1
  state "刷新到磁盘" as S2
  state "校验 sequence" as S3
  state "校验 tool 配对" as S4
  state "发现损坏" as S5
  state "追加修复消息" as S6
  state "恢复完成" as S7
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
type TranscriptRecord = {
  version: 1
  sessionId: string
  sequence: number
  messageId: string
  message: Message
  sha256: string
  writtenAt: string
}

type Message = UserMessage | AssistantMessage | ToolResultMessage | SystemRepairMessage

type ToolResultMessage = {
  id: string
  role: "tool"
  toolUseId: string
  isError: boolean
  content: Array<{ type: "text"; text: string } | { type: "external"; path: string; preview: string }>
}
~~~

## 默认值

| 参数名 | 中文含义 | 单位 | 默认值 | 为什么是这个值 | 触发行为 | 调大后果 | 调小后果 |
|---|---|---|---:|---|---|---|---|
| `transcriptVersion` | 当前 JSONL record 版本 | 配置值 | `1` | 该值是第一版实现的固定边界，用来保证行为可预测、可测试、可回放。 | 对应模块执行到该决策点时读取。 | 边界更宽松，但成本、延迟或风险会上升。 | 边界更保守，但更容易提前截断、阻塞或要求确认。 |
| `maxInlineToolResultChars` | 超过后写外部文件 | 字符 | `50_000` | 这个大小限制用于防止单个输出、文件或事件占满上下文和磁盘预算。 | 内容大小超过该值时截断、落盘、拒绝或只保留预览。 | 能保留更多原始内容，但上下文和存储压力更大。 | 系统更轻，但模型可直接看到的信息更少。 |
| `fsyncEveryRecords` | 第一版每条消息落盘 | 个 | `1` | 这个数量限制用于控制一次任务的执行规模，防止循环、并发或子任务无限扩张。 | 数量达到该值时停止、排队、截断或要求用户确认。 | 吞吐和覆盖面更高，但成本、冲突和排查难度上升。 | 系统更稳，但复杂任务更容易分多轮完成。 |
| `repairMessageRole` | 修复记录用 system repair message | 文本/策略 | `system` | 这个值按 token 预算设置，用来把关键上下文放进模型输入，同时避免某一类内容挤掉最新用户意图。 | 组装 prompt 或计算 token 预算时使用。 | 该类内容可保留更多，但会挤压其它上下文。 | prompt 更紧凑，但可能丢失必要背景。 |
| `hashAlgorithm` | 校验 transcript record | 文本/策略 | `sha256` | 该值是第一版实现的固定边界，用来保证行为可预测、可测试、可回放。 | 对应模块执行到该决策点时读取。 | 边界更宽松，但成本、延迟或风险会上升。 | 边界更保守，但更容易提前截断、阻塞或要求确认。 |

## 详细流程

1. append 前根据当前文件尾部 sequence 生成下一序号。
2. 序列化 message，计算 sha256。
3. 写入 JSONL 单行，立刻 flush。
4. 如果是 assistant，提取 tool_use 顺序写入 metadata.pendingToolUses。
5. 如果是 tool_result，必须匹配 pendingToolUses[0]。
6. 工具输出过大时先写 tool-results 文件，再写 external 引用。
7. resume 时从头读取 JSONL，验证 sequence 连续和 hash 正确。
8. 发现最后一行半写入时丢弃最后一行并追加 repair message。

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| hash_mismatch | 阻塞自动恢复，要求用户确认是否信任。 |
| sequence_gap | 从缺口处停止恢复，追加 repair message。 |
| missing_tool_result | 合成错误 tool_result，内容说明工具结果丢失。 |
| external_output_missing | 保留 preview，标记 missingExternalOutput=true。 |
| json_parse_last_line_failed | 丢弃最后一行，其它行继续恢复。 |

## 提示词模板

本章不使用模型提示词；损坏修复由确定性 repair message 完成。

## 可实现伪代码

~~~ts
function validateTranscript(records: TranscriptRecord[]): TranscriptValidationResult {
  const pending: string[] = []
  for (let i = 0; i < records.length; i++) {
    const record = records[i]
    if (record.sequence !== i + 1) return { ok: false, code: "sequence_gap", index: i }
    if (sha256(record.message) !== record.sha256) return { ok: false, code: "hash_mismatch", index: i }

    if (record.message.role === "assistant") {
      for (const block of record.message.content) {
        if (block.type === "tool_use") pending.push(block.id)
      }
    }

    if (record.message.role === "tool") {
      const expected = pending.shift()
      if (!expected) return { ok: false, code: "orphan_tool_result", index: i }
      if (expected !== record.message.toolUseId) return { ok: false, code: "tool_order_mismatch", expected, actual: record.message.toolUseId }
    }
  }
  if (pending.length) return { ok: false, code: "missing_tool_result", pending }
  return { ok: true }
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 顺序正确 | assistant(toolu_1,toolu_2) 后接两个结果 | 校验通过。 |
| 结果错序 | toolu_2 在 toolu_1 前 | tool_order_mismatch。 |
| 缺结果 | assistant 后无 tool_result | missing_tool_result。 |
| 大输出 | 工具输出 80_000 字符 | 主 transcript 只有 external 引用。 |

## 验收标准

- transcript 可完整恢复为 provider 输入。
- 损坏时有 repair message。
- 大输出不会进入主上下文。
- 配对校验是确定性的。
