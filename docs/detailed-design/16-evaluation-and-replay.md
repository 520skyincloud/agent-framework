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
| `defaultReplayTimeoutMs` | 单个 replay case 超时 | 毫秒 | `120_000` | replay 应快速发现回归，2 分钟足够覆盖普通主循环和工具模拟。 | 单 case 超过 120 秒时标记 timeout failure。 | 慢用例更少误失败，但 CI 等待更久。 | 慢回归更快暴露，但复杂用例可能误失败。 |
| `maxReplayTurns` | replay 默认 turn 上限 | 轮 | `20` | replay 应复现一次用户请求的完整循环，和 runtime 默认 `maxTurns` 保持一致。 | 回放超过 20 轮时失败并输出最后状态。 | 可覆盖更长任务，但 CI 成本增加。 | 长任务 fixture 需要拆分。 |
| `fixtureVersion` | replay fixture 格式版本 | 版本号 | `1` | v1 固定输入、期望事件、模拟工具结果和断言结构。 | 读取 fixture 时按 version 解析；未知版本要求迁移。 | 新版本可表达更多断言，但要维护迁移。 | 无版本会让旧 fixture 语义不清。 |
| `allowNetwork` | replay 是否允许真实网络 | 布尔 | `false` | replay 必须可重复，默认禁止真实网络请求。 | 工具模拟层发现网络访问时返回 blocked_by_replay。 | 若开启，可测真实集成但结果不稳定。 | 保持关闭需要为网络工具准备 fixture。 |
| `allowRealWrite` | replay 是否允许真实写工作区 | 布尔 | `false` | 回放测试不能污染开发者工作区，默认只写临时沙箱。 | Edit/Write/Bash 写操作被重定向到 fixture sandbox。 | 若开启，更接近真实环境但可能破坏文件。 | 保持关闭会要求更多沙箱映射。 |

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
