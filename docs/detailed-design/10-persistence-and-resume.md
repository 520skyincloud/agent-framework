# 持久化与恢复

## 设计目标

- 定义会话如何落盘、进程崩溃后如何恢复、恢复后第一轮如何继续。

## 非目标

- 不接管其它专题的职责。
- 不使用隐藏全局状态。
- 不把失败留给调用方猜测。

## 核心规则

- session 目录必须可迁移。
- metadata 和 transcript 分开。
- 恢复时先校验 transcript，再恢复 tasks。
- running task 超过心跳窗口标记 unknown。
- 恢复后如果上下文超 blockAt，必须先 compact。

## 状态机

~~~mermaid
stateDiagram-v2
  state "读取 metadata" as S0
  state "读取 transcript" as S1
  state "校验 hash 和 sequence" as S2
  state "校验工具配对" as S3
  state "检查外部输出" as S4
  state "恢复任务状态" as S5
  state "恢复子 Agent" as S6
  state "重新估算 token" as S7
  state "生成 ResumeReport" as S8
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
type ResumeReport = { ok: boolean; sessionId: string; messages: Message[]; staleTasks: string[]; missingOutputs: string[]; needsCompactBeforeNextModel: boolean }
~~~

## 默认值

| 配置 | 默认值 | 说明 |
|---|---:|---|
| sessionRoot | ~/.agent/sessions | 默认会话目录。 |
| metadataFlushMs | 500 | metadata 刷盘间隔。 |
| taskHeartbeatStaleMs | 30_000 | 任务心跳过期阈值。 |
| maxSessionBytes | 524_288_000 | 单会话 500MB 上限。 |
| resumeRequiresValidation | true | 恢复必须校验。 |

## 详细流程

1. 读取 metadata.json。
2. 读取 transcript.jsonl 并校验。
3. 加载 tool-results 索引。
4. 加载 tasks 状态；running 且心跳过期标 unknown。
5. 加载 subagent metadata。
6. 重新估算 token。
7. 如果 prompt 超 blockAt，设置 needsCompactBeforeNextModel=true。
8. 返回 ResumeReport。

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| metadata_missing | 从 transcript 重建最小 metadata。 |
| transcript_corrupt | 追加 repair message 或阻塞。 |
| tool_output_missing | 标记缺失，保留 preview。 |
| task_stale | 标记 unknown，不假设成功。 |
| session_too_large | 阻塞新工具输出，要求清理。 |

## 提示词模板

本章没有默认模型调用；如果实现需要模型参与，必须复用上下文压缩、权限解释或工具错误恢复章节的固定提示词。

## 可实现伪代码

~~~ts
async function resumeSession(sessionId: string): Promise<ResumeReport> {
  const metadata = await loadMetadataOrRebuild(sessionId)
  const transcript = await loadAndValidateTranscript(sessionId)
  const missingOutputs = await verifyToolOutputs(transcript.messages)
  const staleTasks = await markStaleTasks(metadata.tasks, 30_000)
  const tokenEstimate = estimateTokens(transcript.messages)
  const needsCompact = tokenEstimate > metadata.context.blockAt
  return { ok: transcript.ok, sessionId, messages: transcript.messages, staleTasks, missingOutputs, needsCompactBeforeNextModel: needsCompact }
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 正常恢复 | 完整 metadata + transcript | ResumeReport.ok=true。 |
| metadata 缺失 | 只有 transcript.jsonl | 重建最小 metadata。 |
| 最后一行损坏 | JSONL 最后一行半写入 | 丢弃最后一行并追加 repair message。 |
| 任务心跳过期 | running task heartbeat 超过 30_000 ms | 标记 unknown。 |
| 恢复后超上下文 | tokenEstimate > blockAt | needsCompactBeforeNextModel=true。 |

## 验收标准

- 有具体默认值。
- 有结构化错误码。
- 有可执行伪代码。
- 测试覆盖正常路径、失败路径和边界值。
