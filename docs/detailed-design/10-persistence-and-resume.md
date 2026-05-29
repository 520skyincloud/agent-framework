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

| 参数名 | 中文含义 | 单位 | 默认值 | 为什么是这个值 | 触发行为 | 调大后果 | 调小后果 |
|---|---|---|---:|---|---|---|---|
| `sessionRoot` | 默认会话目录 | 路径 | `~/.agent/sessions` | 会话、transcript、工具结果和任务状态集中放在一个可备份目录，便于恢复和清理。 | 创建或恢复 session 时从该目录读取 metadata 和 transcript。 | 改到项目内更易随仓库移动，但可能污染 repo。 | 改到临时目录会让恢复能力变弱。 |
| `metadataFlushMs` | metadata 刷盘间隔 | 毫秒 | `500` | metadata 记录 pending tool、任务心跳和预算状态，500ms 能减少崩溃丢状态。 | metadata 变更后最多 500ms 内写盘。 | 写盘更少、性能更好，但崩溃后状态更旧。 | 写盘更频繁，磁盘开销更高。 |
| `taskHeartbeatStaleMs` | 任务心跳过期阈值 | 毫秒 | `30_000` | 后台任务 30 秒无心跳通常说明进程退出、挂起或宿主重启。 | resume 时发现 running task 心跳超过 30 秒未更新，标记为 unknown。 | 慢任务更少误判 stale，但恢复更晚发现失联。 | 短暂卡顿也可能被标记 unknown。 |
| `maxSessionBytes` | 单会话存储上限 | 字节 | `524_288_000` | 500MB 能容纳长任务 transcript 和工具输出，同时防止磁盘无限增长。 | 写入会话文件前统计目录大小，超过后要求清理或归档。 | 长任务保留更多原始证据，但磁盘占用上升。 | 更早需要落盘清理、压缩或新建会话。 |
| `resumeRequiresValidation` | 恢复前是否强制校验 | 布尔 | `true` | 恢复前必须校验 hash、tool_use/tool_result 配对和任务状态，避免把损坏历史发给模型。 | resumeSession 先执行 validation，不通过则返回 repair/block。 | 若关闭，恢复更快但可能带着坏 transcript 继续。 | 保持开启会多一步检查，但结果可信。 |

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

来源：不适用。本章没有默认模型调用；如果实现需要模型参与，必须复用上下文压缩、权限解释或工具错误恢复章节的固定提示词。

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
