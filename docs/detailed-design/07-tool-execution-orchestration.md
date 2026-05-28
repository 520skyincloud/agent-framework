# 工具执行编排

## 设计目标

- 把同一 assistant 消息里的多个 tool_use 拆成安全批次，并保证返回顺序不变。

## 非目标

- 不接管其它专题的职责。
- 不使用隐藏全局状态。
- 不把失败留给调用方猜测。

## 核心规则

- Read/Grep/Glob 默认并发。
- Edit/Write 默认串行。
- Bash 只有只读命令可并发。
- 同一路径的写操作必须串行。
- 最终 tool_result 顺序必须等于 tool_use 顺序。
- 并发上限默认 10，MVP 建议 4。

## 状态机

~~~mermaid
stateDiagram-v2
  state "接收多个 tool_use" as S0
  state "标注安全等级" as S1
  state "生成执行计划" as S2
  state "并发执行只读批次" as S3
  state "串行执行写入工具" as S4
  state "收集结果" as S5
  state "按原始顺序排序" as S6
  state "补齐中断结果" as S7
  state "返回 tool_result 列表" as S8
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
type ToolPlanStep = { kind: "parallel" | "serial"; toolUseIds: string[]; reason: string }
type ToolBatchResult = { orderedMessages: ToolResultMessage[]; plan: ToolPlanStep[]; errors: ToolExecutionError[] }
~~~

## 默认值

| 参数名 | 中文含义 | 单位 | 默认值 | 为什么是这个值 | 触发行为 | 调大后果 | 调小后果 |
|---|---|---|---:|---|---|---|---|
| `maxConcurrency` | 成熟版工具并发上限 | 个 | `10` | 10 能提升只读工具吞吐，同时不让 IO 和 provider 事件过载。 | 并发执行只读工具批次时使用。 | 更快但更容易抢资源和乱序。 | 更稳定但速度慢。 |
| `mvpMaxConcurrency` | 第一版工具并发上限 | 个 | `4` | 4 更适合 MVP，容易观察和调试。 | 第一版工具编排默认使用。 | 吞吐更高但调试更难。 | 吞吐更低但更稳。 |
| `unsafeSerial` | 不安全工具串行 | 文本/策略 | `true` | 这个策略值用于给副作用、安全或部署行为一个确定默认，避免实现时出现隐式放行。 | 权限、部署或安全检查进入对应分支时使用。 | 如果改得更宽松，操作更顺滑但安全风险更高。 | 如果改得更保守，安全性更强但需要更多确认。 |
| `samePathWriteSerial` | 同一路径写入串行 | 文本/策略 | `true` | 这个策略值用于给副作用、安全或部署行为一个确定默认，避免实现时出现隐式放行。 | 权限、部署或安全检查进入对应分支时使用。 | 如果改得更宽松，操作更顺滑但安全风险更高。 | 如果改得更保守，安全性更强但需要更多确认。 |
| `batchTimeoutMs` | 一批工具最大时间 | 毫秒 | `600_000` | 这个时间值用于区分正常等待和疑似卡死，第一版优先保证任务不会无限挂起。 | 运行时间或等待时间超过该值时触发超时、刷新、心跳或清理。 | 更不容易误杀慢任务，但卡住时等待更久。 | 故障暴露更快，但慢任务更容易被误判。 |

## 详细流程

1. 分析每个 tool_use 的 safety：readOnly、writePath、longRunning。
2. 连续 readOnly 工具组成并发批次。
3. 遇到 Edit/Write/危险 Bash，关闭当前批次并单独执行。
4. 执行结果按 toolUseIndex 存入 map。
5. 所有批次完成后按原始索引输出。
6. 批次中断时为未执行工具生成 aborted result。

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| one_tool_failed | 其它已启动安全工具继续；危险串行工具失败后停止后续危险工具。 |
| batch_timeout | 取消未完成工具并合成 timeout。 |
| ordering_bug | 阻塞发送 provider，抛出 internal_order_error。 |

## 提示词模板

本章没有默认模型调用；如果实现需要模型参与，必须复用上下文压缩、权限解释或工具错误恢复章节的固定提示词。

## 可实现伪代码

~~~ts
async function runToolBatch(input: ToolBatchInput): Promise<ToolBatchResult> {
  const plan = planToolExecution(input.toolUses, input.registry)
  const byId = new Map<string, ToolRunResult>()
  for (const step of plan) {
    const runs = step.kind === "parallel" ? await runParallel(step.toolUseIds, input.maxConcurrency) : await runSerial(step.toolUseIds)
    for (const result of runs) byId.set(result.toolUseId, result)
  }
  const orderedMessages = input.toolUses.map(tu => byId.get(tu.id)?.message ?? abortedToolResult(tu))
  return { orderedMessages, plan, errors: collectErrors(orderedMessages) }
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 只读并发 | Read(a)、Grep(foo)、Glob(*) | 同一 parallel batch，并发数不超过 10。 |
| 写入串行 | Read(a)、Edit(a)、Read(b) | Read(a) 先执行，Edit 单独串行，Read(b) 可在后批次。 |
| 同路径写入 | Edit(a)、Write(a) | 两个写操作串行。 |
| 并发结果乱序 | Grep 先完成、Read 后完成 | 返回给模型仍按 Read、Grep 顺序。 |
| 批次超时 | parallel batch 超过 600_000 ms | 未完成工具得到 timeout tool_result。 |

## 验收标准

- 有具体默认值。
- 有结构化错误码。
- 有可执行伪代码。
- 测试覆盖正常路径、失败路径和边界值。
