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
| `transcriptVersion` | 当前 JSONL 记录格式版本 | 版本号 | `1` | v1 固定 role、content、toolUseId、sequence 和 hash 字段，先保证可恢复和可回放。 | 读取 transcript 时按 version 选择解析器；未知版本直接停止恢复并提示迁移。 | 版本升级可支持新字段，但需要迁移和兼容测试。 | 不做版本号会让旧会话无法安全识别格式。 |
| `maxInlineToolResultChars` | 工具结果内联字符上限 | 字符 | `50_000` | 50k 以上工具结果会明显挤占模型输入，完整内容应转为外部文件引用。 | tool_result 超过 50k 字符时写入 `tool-results`，消息中保留 path 和 preview。 | 模型可直接看到更多原始输出，但 transcript 和 prompt 更臃肿。 | 更多工具结果需要二次读取，模型即时可见信息减少。 |
| `fsyncEveryRecords` | 每写多少条 record 刷盘一次 | 条 | `1` | 每条消息都刷盘能最大化崩溃恢复可靠性，第一版优先保证 transcript 不丢。 | append 一条 JSONL 后立即 flush/fsync。 | 批量刷盘吞吐更高，但崩溃时可能丢最后几条。 | 低于 1 无意义；保持 1 是最安全边界。 |
| `repairMessageRole` | 修复记录使用的消息角色 | 角色名 | `system` | 修复消息是框架写入的恢复说明，不应伪装成用户或助手。 | 需要补齐工具结果或解释 transcript 修复时写 system repair message。 | 改成 assistant 会污染模型行为记录。 | 改成 user 会让后续模型误以为用户下过该指令。 |
| `hashAlgorithm` | transcript record 校验算法 | 算法名 | `sha256` | sha256 普遍可用，足以检测 record 被截断或篡改。 | append 时计算 hash，resume 时重新计算并比对。 | 换更强算法收益有限但兼容成本上升。 | 换弱算法会降低损坏检测可信度。 |

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
