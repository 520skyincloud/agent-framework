# 评估与回放

## 设计目标

- 用固定模型和固定工具回放历史 case，防止重构破坏行为。

## 非目标

- 不接管其它专题的职责。
- 不使用隐藏全局状态。
- 不把失败留给调用方猜测。

## 核心规则

- replay 禁止真实网络和真实写文件，除非 fixture 显式允许。
- fake model 必须按 turn 返回固定 assistant。
- fake tools 必须按 toolUseId 返回固定结果。
- 断言事件序列、transcript、文件 diff。
- 第一处差异必须可读。

## 状态机

~~~mermaid
stateDiagram-v2
  state "读取 fixture" as S0
  state "创建临时工作区" as S1
  state "注入假模型" as S2
  state "注入假工具" as S3
  state "运行主循环" as S4
  state "收集事件" as S5
  state "比对 transcript" as S6
  state "比对文件" as S7
  state "输出报告" as S8
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
type ReplayCase = { version: 1; name: string; inputMessages: Message[]; modelResponses: AssistantMessage[]; toolResults: Record<string, ToolResultMessage>; expectedEvents: Partial<RuntimeEvent>[]; expectedFiles?: Record<string,string> }
~~~

## 默认值

| 参数名 | 中文含义 | 单位 | 默认值 | 为什么是这个值 | 触发行为 | 调大后果 | 调小后果 |
|---|---|---|---:|---|---|---|---|
| `defaultReplayTimeoutMs` | 单 case 超时 | 毫秒 | `120_000` | 这个时间值用于区分正常等待和疑似卡死，第一版优先保证任务不会无限挂起。 | 运行时间或等待时间超过该值时触发超时、刷新、心跳或清理。 | 更不容易误杀慢任务，但卡住时等待更久。 | 故障暴露更快，但慢任务更容易被误判。 |
| `maxReplayTurns` | 默认 turn 上限 | 轮 | `20` | 这个数量限制用于控制一次任务的执行规模，防止循环、并发或子任务无限扩张。 | 数量达到该值时停止、排队、截断或要求用户确认。 | 吞吐和覆盖面更高，但成本、冲突和排查难度上升。 | 系统更稳，但复杂任务更容易分多轮完成。 |
| `fixtureVersion` | fixture 版本 | 配置值 | `1` | 该值是第一版实现的固定边界，用来保证行为可预测、可测试、可回放。 | 对应模块执行到该决策点时读取。 | 边界更宽松，但成本、延迟或风险会上升。 | 边界更保守，但更容易提前截断、阻塞或要求确认。 |
| `allowNetwork` | 默认禁网 | 文本/策略 | `false` | 这个策略值用于给副作用、安全或部署行为一个确定默认，避免实现时出现隐式放行。 | 权限、部署或安全检查进入对应分支时使用。 | 如果改得更宽松，操作更顺滑但安全风险更高。 | 如果改得更保守，安全性更强但需要更多确认。 |
| `allowRealWrite` | 默认禁真实写 | 文本/策略 | `false` | 这个策略值用于给副作用、安全或部署行为一个确定默认，避免实现时出现隐式放行。 | 权限、部署或安全检查进入对应分支时使用。 | 如果改得更宽松，操作更顺滑但安全风险更高。 | 如果改得更保守，安全性更强但需要更多确认。 |

## 详细流程

1. 读取 replay case。
2. 创建临时 workspace。
3. 注入 fake model。
4. 注入 fake tool registry。
5. 运行 queryLoop。
6. 收集事件和 transcript。
7. 比对 expectedEvents、expectedToolCalls、expectedFiles。
8. 输出第一处 mismatch。

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| fixture_invalid | 拒绝运行。 |
| unexpected_tool_call | 失败并显示 tool name/input。 |
| event_mismatch | 显示 expected/actual。 |
| file_diff_mismatch | 输出 unified diff。 |
| timeout | 保存 partial replay report。 |

## 提示词模板

本章没有默认模型调用；如果实现需要模型参与，必须复用上下文压缩、权限解释或工具错误恢复章节的固定提示词。

## 可实现伪代码

~~~ts
async function runReplayCase(testCase: ReplayCase): Promise<ReplayReport> {
  const workspace = await createTempWorkspace(testCase)
  const model = new FakeModel(testCase.modelResponses)
  const tools = new FakeToolRegistry(testCase.toolResults)
  const events = await collect(queryLoop({ sessionId: workspace.sessionId, userMessage: lastUser(testCase), route: model.route, tools: tools.all(), maxTurns: 20 }))
  return compareReplay({ events, workspace }, testCase)
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 固定模型响应 | fake model 两轮 assistant | 主循环事件与 expectedEvents 一致。 |
| 意外工具调用 | 模型调用 fixture 未声明工具 | unexpected_tool_call。 |
| 文件 diff 不同 | 期望 a.ts 内容 X，实际 Y | 输出 unified diff。 |
| 禁网 | 工具尝试真实网络 | replay 拒绝。 |
| 超时 | case 超过 120_000 ms | 保存 partial report。 |

## 验收标准

- 有具体默认值。
- 有结构化错误码。
- 有可执行伪代码。
- 测试覆盖正常路径、失败路径和边界值。
