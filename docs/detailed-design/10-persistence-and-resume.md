# 持久化与恢复

## 设计目标

定义 session、metadata、transcript、工具输出、任务状态、恢复算法和损坏修复。

本章必须写到工程师可以直接实现：输入、输出、状态、默认数字、失败处理和验收方式都要明确。

## 非目标

- 不接管其它专题的职责。
- 不用隐藏全局状态传递关键数据。
- 不用自然语言错误代替结构化错误。
- 不把“后续再定”当作实现方案。

## 核心规则

- 所有输入必须显式传入。
- 所有输出必须能被 UI、SDK、replay 共用。
- 所有失败必须返回结构化错误：code、message、recoverable、nextAction。
- 任何影响下一轮模型输入的状态都必须进入 transcript 或 metadata。
- 任何副作用都必须先过权限系统和预算系统。
- 默认值必须集中在配置或常量模块。

## 状态机

~~~mermaid
stateDiagram-v2
  state "接收请求" as Receive
  state "校验输入" as Validate
  state "检查预算权限" as Guard
  state "执行核心逻辑" as Execute
  state "持久化结果" as Persist
  state "发出事件" as Emit
  state "成功" as Success
  state "失败" as Failure

  [*] --> Receive
  Receive --> Validate
  Validate --> Guard: 输入合法
  Validate --> Failure: 输入非法
  Guard --> Execute: 允许执行
  Guard --> Failure: 被拒绝
  Execute --> Persist
  Execute --> Failure: 执行失败
  Persist --> Emit
  Emit --> Success
  Success --> [*]
  Failure --> [*]
~~~

## 数据结构

~~~ts
type ResumeSessionInput = {
  sessionId: string
  agentId?: string
  requestId: string
  cwd: string
  config: RuntimeConfig
  state: RuntimeState
}

type ResumeSessionResult =
  | { ok: true; events: RuntimeEvent[]; statePatch?: Partial<RuntimeState>; messages?: Message[] }
  | { ok: false; events: RuntimeEvent[]; error: RuntimeError }

type RuntimeError = {
  code: string
  message: string
  recoverable: boolean
  nextAction: "retry" | "compact" | "ask_user" | "abort" | "fallback"
  details?: Record<string, unknown>
}
~~~

## 默认值

| 配置 | 默认值 | 说明 |
|---|---:|---|
| requestTimeoutMs | 120_000 | 单次请求超时。 |
| maxRetries | 2 | 模块内部恢复重试。 |
| eventFlushMs | 100 | 事件刷新间隔。 |
| persistRequired | true | 影响后续模型的状态必须持久化。 |

## 详细流程

1. 读取 metadata。
2. 回放 transcript。
3. 校验消息 id 和工具配对。
4. 检查外部工具输出文件。
5. 修复陈旧任务状态。
6. 重新估算 token。
7. 超阻塞阈值则先 compact。

## 失败处理

| 失败 | 处理 |
|---|---|
| 输入缺字段 | 返回 invalid_input，指出缺失字段，不执行核心逻辑。 |
| 权限拒绝 | 返回 permission_denied，不自动重试，可让用户确认。 |
| 超预算 | 返回 budget_exceeded，附当前预算和需要的预算。 |
| 执行超时 | 返回 timeout，保留已产生事件。 |
| 持久化失败 | 返回 persistence_failed，阻塞继续执行。 |
| replay 不一致 | 返回 replay_mismatch，输出第一处不同事件。 |

## 提示词模板

本章默认不需要专用模型提示词。若实现中需要模型参与，必须把输入、输出格式、失败分支写成固定模板。

## 可实现伪代码

~~~ts
async function resumeSession(input: ResumeSessionInput): Promise<ResumeSessionResult> {
  const events: RuntimeEvent[] = []
  const validation = validateInput(input)
  if (!validation.ok) return { ok: false, events, error: validation.error }

  const guard = await checkBudgetAndPermission(input)
  if (!guard.ok) {
    events.push({ type: "guard_rejected", requestId: input.requestId, code: guard.error.code })
    return { ok: false, events, error: guard.error }
  }

  try {
    events.push({ type: "module_started", requestId: input.requestId })
    const result = await executeCore(input, events)
    await persistResult(input.sessionId, result)
    events.push({ type: "module_completed", requestId: input.requestId })
    return { ok: true, events, statePatch: result.statePatch, messages: result.messages }
  } catch (error) {
    const normalized = normalizeRuntimeError(error)
    events.push({ type: "module_failed", requestId: input.requestId, code: normalized.code })
    return { ok: false, events, error: normalized }
  }
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 正常路径 | 合法输入 | ok=true。 |
| 输入缺失 | 缺 sessionId | invalid_input。 |
| 权限拒绝 | deny 命中 | permission_denied。 |
| 超预算 | 预算不足 | budget_exceeded。 |
| 持久化失败 | transcript 不可写 | persistence_failed。 |

## 验收标准

- 正常路径、失败路径、边界值都有自动化测试。
- 所有错误都有 code 和 nextAction。
- 影响下一轮模型行为的数据已经持久化。
- replay 可以复现关键事件序列。
- 文档中的默认数字能在配置或常量模块找到同名字段。
