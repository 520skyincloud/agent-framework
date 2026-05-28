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
| `eventSchemaVersion` | 事件结构版本 | 版本号 | `1` | v1 固定事件类型、时间戳、sessionId、spanId 和 payload 形状，便于 UI 和 replay 共用。 | 发出事件时写入 version；消费者遇到未知版本必须降级或拒绝。 | 新版本可扩展字段，但要维护兼容层。 | 没有版本会让旧 UI 无法识别新事件。 |
| `eventFlushMs` | 事件刷新间隔 | 毫秒 | `100` | 100ms 能平衡流式体验和 UI 渲染压力。 | 事件缓冲超过 100ms 时批量发送。 | 批次更大、性能更好，但 UI 反馈变慢。 | UI 更实时，但事件数量和渲染成本上升。 |
| `maxEventBytes` | 单事件最大大小 | 字节 | `65_536` | 64KB 能容纳错误摘要和工具状态，但不允许把长日志塞进事件。 | 单事件超过 64KB 时截断 payload，并把完整内容写外部文件。 | 可内联更多细节，但事件流和前端内存压力增加。 | 更早外部化，排查时需要跳转文件。 |
| `auditRetentionDays` | 审计事件保留天数 | 天 | `30` | 30 天覆盖近期权限、部署和安全问题排查。 | 清理任务删除超过 30 天的 audit event。 | 可追溯更久，但存储和隐私压力上升。 | 事故排查时间窗口缩短。 |
| `redactSecrets` | 是否默认脱敏敏感字段 | 布尔 | `true` | 事件会进入日志、UI 和 replay，默认必须隐藏 token、key、密码和 cookie。 | 事件写入前对匹配 secret 规则的字段脱敏。 | 若关闭，调试更直接但泄密风险高。 | 保持开启可能隐藏少量排查细节。 |

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
