# 运行时主循环

## 设计目标

- 把一次用户输入推进到终止状态：模型输出、工具执行、继续循环或停止。
- 让主循环成为 async generator，UI、SDK、CLI、replay 都消费同一事件流。
- 保证任何退出路径都不会留下孤立 tool_use。

## 非目标

- 不在主循环内实现具体工具逻辑。
- 不在主循环内写权限规则。
- 不在主循环内直接拼 provider 私有格式。

## 核心规则

- 每轮模型调用前必须先写入用户消息、metadata 和上下文预算事件。
- assistant 消息必须先完整写入 transcript，再执行其中的 tool_use。
- 工具结果必须按 assistant 消息中 tool_use 的原始顺序返回给模型。
- 默认 maxTurns=20；达到后返回 done:max_turns。
- abort 时必须为未完成 tool_use 合成错误 tool_result。
- 模型调用、工具执行、持久化、权限拒绝都必须发 RuntimeEvent。

## 状态机

~~~mermaid
stateDiagram-v2
  state "接收用户输入" as S0
  state "持久化用户消息" as S1
  state "检查上下文预算" as S2
  state "组装 Prompt" as S3
  state "调用模型" as S4
  state "写入 assistant" as S5
  state "判断工具" as S6
  state "执行工具" as S7
  state "写入工具结果" as S8
  state "校验配对" as S9
  state "完成或继续" as S10
  [*] --> S0
  S0 --> S1
  S1 --> S2
  S2 --> S3
  S3 --> S4
  S4 --> S5
  S5 --> S6
  S6 --> S7
  S7 --> S8
  S8 --> S9
  S9 --> S10
  S10 --> [*]
~~~

## 数据结构

~~~ts
type QueryLoopInput = {
  sessionId: string
  userMessage: UserMessage
  route: ModelRoute
  tools: ToolDefinition[]
  maxTurns?: number
  abortSignal: AbortSignal
}

type QueryLoopStopReason = "stop" | "max_turns" | "aborted" | "context_blocked" | "model_error" | "tool_error"

type RuntimeEvent =
  | { type: "turn_started"; sessionId: string; turn: number }
  | { type: "context_decision"; action: ContextBudgetDecision["action"]; promptTokens: number }
  | { type: "model_stream_start"; requestId: string; model: string }
  | { type: "assistant_delta"; text: string }
  | { type: "assistant_message"; messageId: string }
  | { type: "tool_started"; toolUseId: string; name: string }
  | { type: "tool_finished"; toolUseId: string; ok: boolean; durationMs: number }
  | { type: "done"; reason: QueryLoopStopReason }
~~~

## 默认值

| 配置 | 默认值 | 说明 |
|---|---:|---|
| maxTurns | 20 | 一次用户请求最多自循环 20 轮。 |
| turnHardTimeoutMs | 1_800_000 | 单次用户请求硬超时 30 分钟。 |
| eventFlushMs | 100 | UI 事件刷新间隔。 |
| abortRepairRequired | true | 中断时补齐错误 tool_result。 |
| pairValidation | after_each_tool_batch | 每批工具结果后校验配对。 |

## 详细流程

1. 创建 state：turn=0、terminalReason=null。
2. 写入 user message；如果写入失败，直接返回 persistence_failed。
3. 进入 while turn < maxTurns。
4. 调用 decideContextAction；返回 block 时不调用模型。
5. assemblePrompt 得到 provider 输入，并记录 promptTokens。
6. 调用 streamModel；每个 delta 都发 assistant_delta。
7. 流结束后构造 assistant message，写入 transcript。
8. 如果 assistant 没有 tool_use，执行 stop hooks，返回 done:stop。
9. 如果有 tool_use，交给 runToolBatch；按原始顺序写入 tool_result。
10. 调用 validateToolPairsOrThrow；通过后 turn++，继续下一轮。

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| transcript_write_failed | 不可恢复，停止主循环，返回 done:model_error 或 done:tool_error。 |
| context_blocked | 不调用模型，返回 done:context_blocked，并附 compact 建议。 |
| model_stream_idle | 交给模型路由层重试；重试失败返回 done:model_error。 |
| invalid_tool_use | 写 repair message，请模型修复一次；再次失败终止。 |
| abort_during_tools | 合成 isError=true 的 tool_result，然后 done:aborted。 |

## 提示词模板

本章不直接调用模型；模型修复提示词放在工具协议和模型路由章节。

## 可实现伪代码

~~~ts
async function* queryLoop(input: QueryLoopInput): AsyncGenerator<RuntimeEvent> {
  const maxTurns = input.maxTurns ?? 20
  await transcript.append(input.sessionId, input.userMessage)

  for (let turn = 0; turn < maxTurns; turn++) {
    yield { type: "turn_started", sessionId: input.sessionId, turn }

    const messages = await transcript.loadMessages(input.sessionId)
    const context = decideContextAction({ messages, route: input.route })
    yield { type: "context_decision", action: context.action, promptTokens: context.promptTokens }

    if (context.action === "block") {
      yield { type: "done", reason: "context_blocked" }
      return
    }
    if (context.action !== "continue") await runCompaction(context)

    const request = assemblePrompt({ sessionId: input.sessionId, route: input.route, tools: input.tools })
    yield { type: "model_stream_start", requestId: request.id, model: request.model }

    const assistant = await collectAssistant(request, input.abortSignal)
    await transcript.append(input.sessionId, assistant)
    yield { type: "assistant_message", messageId: assistant.id }

    const toolUses = extractToolUses(assistant)
    if (toolUses.length === 0) {
      yield { type: "done", reason: "stop" }
      return
    }

    const results = await runToolBatch({ sessionId: input.sessionId, toolUses, tools: input.tools, abortSignal: input.abortSignal })
    for (const result of orderToolResults(results, toolUses)) {
      await transcript.append(input.sessionId, result.message)
      yield { type: "tool_finished", toolUseId: result.toolUseId, ok: result.ok, durationMs: result.durationMs }
    }
    validateToolPairsOrThrow(await transcript.loadMessages(input.sessionId))
  }

  yield { type: "done", reason: "max_turns" }
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 无工具结束 | 模型返回纯文本 | 事件最后是 done:stop。 |
| 两个工具调用 | assistant 包含 toolu_1、toolu_2 | tool_result 顺序必须是 1、2。 |
| 工具中断 | abortSignal 在第二个工具前触发 | 第二个工具得到错误 tool_result。 |
| 上下文阻塞 | promptTokens > blockAt 且 compact 失败 | 不调用模型，done:context_blocked。 |

## 验收标准

- 主循环是 async generator。
- 所有退出路径都有 done 事件。
- 没有孤立 tool_use。
- 达到 maxTurns=20 必须停止。
