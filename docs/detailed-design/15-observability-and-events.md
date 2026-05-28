# 可观测性与事件

## 设计目标

- 定义事件、指标、trace、审计日志和成本记录，让运行过程可解释。

## 非目标

- 不接管其它专题的职责。
- 不使用隐藏全局状态。
- 不把失败留给调用方猜测。

## 核心规则

- 所有事件必须有 sessionId、requestId、timestamp。
- 敏感字段必须脱敏。
- 模型调用记录 tokens、cost、latency。
- 工具调用记录 duration、permission、outputSize。
- 权限事件保留 30 天。
- 事件 schema 版本化。

## 状态机

~~~mermaid
stateDiagram-v2
  state "创建事件" as S0
  state "脱敏" as S1
  state "校验 schema" as S2
  state "截断大字段" as S3
  state "写事件日志" as S4
  state "更新指标" as S5
  state "推送 UI" as S6
  state "写审计" as S7
  state "失败降级" as S8
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
type RuntimeEvent = { version: 1; type: string; sessionId: string; requestId: string; timestamp: string; data: Record<string, unknown> }
~~~

## 默认值

| 配置 | 默认值 | 说明 |
|---|---:|---|
| eventSchemaVersion | 1 | 事件版本。 |
| eventFlushMs | 100 | 刷新间隔。 |
| maxEventBytes | 65_536 | 单事件最大 64KB。 |
| auditRetentionDays | 30 | 审计保留。 |
| redactSecrets | true | 默认脱敏。 |

## 详细流程

1. 模块创建事件对象。
2. redactEvent 脱敏。
3. 校验 schema。
4. 写 event log。
5. 同步更新 metrics。
6. 如果 UI 订阅，推送简化事件。
7. 事件过大时截断 details 并保留 path。

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| event_too_large | 截断 details。 |
| redaction_failed | 不发原事件，发 redaction_failed。 |
| event_log_write_failed | 不阻塞主循环，但写 stderr fallback。 |
| metric_write_failed | 记录 warning。 |

## 提示词模板

本章没有默认模型调用；如果实现需要模型参与，必须复用上下文压缩、权限解释或工具错误恢复章节的固定提示词。

## 可实现伪代码

~~~ts
function emitRuntimeEvent(event: RuntimeEvent): void {
  const redacted = redactEvent(event)
  const valid = validateEvent(redacted)
  if (!valid.ok) throw new Error("invalid_event")
  eventLog.append(truncateLargeEvent(redacted, 65_536))
  metrics.update(redacted)
  uiBus.publish(toUiEvent(redacted))
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 模型事件 | 一次模型调用完成 | 记录 inputTokens、outputTokens、cost、latency。 |
| 工具事件 | Read 执行完成 | 记录 durationMs、outputBytes、permission。 |
| secret 脱敏 | 事件 data 含 API key | 写入前替换为 [REDACTED]。 |
| 事件过大 | data 超过 65_536 bytes | 截断 details 并记录 truncated=true。 |
| 日志不可写 | event log 写失败 | 主循环不中断，写 fallback warning。 |

## 验收标准

- 有具体默认值。
- 有结构化错误码。
- 有可执行伪代码。
- 测试覆盖正常路径、失败路径和边界值。
