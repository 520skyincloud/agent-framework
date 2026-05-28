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

| 参数名 | 中文含义 | 单位 | 默认值 | 为什么是这个值 | 触发行为 | 调大后果 | 调小后果 |
|---|---|---|---:|---|---|---|---|
| `eventSchemaVersion` | 事件版本 | tokens | `1` | 这个值按 token 预算设置，用来把关键上下文放进模型输入，同时避免某一类内容挤掉最新用户意图。 | 组装 prompt 或计算 token 预算时使用。 | 该类内容可保留更多，但会挤压其它上下文。 | prompt 更紧凑，但可能丢失必要背景。 |
| `eventFlushMs` | 刷新间隔 | 毫秒 | `100` | 这个时间值用于区分正常等待和疑似卡死，第一版优先保证任务不会无限挂起。 | 运行时间或等待时间超过该值时触发超时、刷新、心跳或清理。 | 更不容易误杀慢任务，但卡住时等待更久。 | 故障暴露更快，但慢任务更容易被误判。 |
| `maxEventBytes` | 单事件最大 64KB | 字节 | `65_536` | 这个大小限制用于防止单个输出、文件或事件占满上下文和磁盘预算。 | 内容大小超过该值时截断、落盘、拒绝或只保留预览。 | 能保留更多原始内容，但上下文和存储压力更大。 | 系统更轻，但模型可直接看到的信息更少。 |
| `auditRetentionDays` | 审计保留 | 天 | `30` | 这个策略值用于给副作用、安全或部署行为一个确定默认，避免实现时出现隐式放行。 | 权限、部署或安全检查进入对应分支时使用。 | 如果改得更宽松，操作更顺滑但安全风险更高。 | 如果改得更保守，安全性更强但需要更多确认。 |
| `redactSecrets` | 默认脱敏 | 文本/策略 | `true` | 该值是第一版实现的固定边界，用来保证行为可预测、可测试、可回放。 | 对应模块执行到该决策点时读取。 | 边界更宽松，但成本、延迟或风险会上升。 | 边界更保守，但更容易提前截断、阻塞或要求确认。 |

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
