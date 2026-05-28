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

| 配置 | 默认值 | 说明 |
|---|---:|---|
| defaultReplayTimeoutMs | 120_000 | 单 case 超时。 |
| maxReplayTurns | 20 | 默认 turn 上限。 |
| fixtureVersion | 1 | fixture 版本。 |
| allowNetwork | false | 默认禁网。 |
| allowRealWrite | false | 默认禁真实写。 |

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
