# 多 Agent 编排

## 设计目标

- 定义子 Agent 的启动、隔离、权限继承、结果总结和父子合并协议。

## 非目标

- 不接管其它专题的职责。
- 不使用隐藏全局状态。
- 不把失败留给调用方猜测。

## 核心规则

- 第一版只允许只读 Explore Agent。
- 子 Agent 默认 maxTurns=20。
- fork 型子 Agent 最大 maxTurns=200。
- 每个父 turn 最多启动 3 个子 Agent。
- 子 Agent 不共享父 Agent 可变 state。
- 子 Agent 结果必须总结后再进父上下文。

## 状态机

~~~mermaid
stateDiagram-v2
  state "读取 Agent 定义" as S0
  state "校验工具子集" as S1
  state "检查 spawn 预算" as S2
  state "创建 sidechain" as S3
  state "复制最小上下文" as S4
  state "运行子 Agent" as S5
  state "生成摘要" as S6
  state "父 Agent 合并或拒绝" as S7
  state "记录结果" as S8
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
type SubagentLaunchRequest = { parentSessionId: string; agentType: string; prompt: string; allowedTools: string[]; maxTurns?: number; mode: "sync" | "background" }
type SubagentResult = { agentId: string; transcriptPath: string; summary: string; filesTouched: string[]; status: "succeeded" | "failed" | "partial" }
~~~

## 默认值

| 参数名 | 中文含义 | 单位 | 默认值 | 为什么是这个值 | 触发行为 | 调大后果 | 调小后果 |
|---|---|---|---:|---|---|---|---|
| `defaultSubagentMaxTurns` | 普通子 Agent 最大轮数 | 轮 | `20` | 普通子 Agent 只应解决一个明确子问题，20 轮足够读取、验证和总结。 | 子 Agent 达到 20 轮后停止，返回 partial summary 给父 Agent。 | 子 Agent 独立性更强，但更容易跑偏和消耗预算。 | 父 Agent 需要更频繁拆分任务。 |
| `forkMaxTurns` | fork 型子 Agent 最大轮数 | 轮 | `200` | 200 给长调查/验证任务足够空间，但仍有硬上限。 | 后台或 fork 子 Agent 使用。 | 长任务更完整但成本高。 | 长调查更容易中断，需要父 Agent 重新派发后续任务。 |
| `maxAgentSpawnsPerTurn` | 单轮子 Agent 启动上限 | 个 | `3` | 3 个足够并行调查、实现、验证，避免 agent 爆炸。 | 父 Agent 同一轮启动超过上限时拒绝。 | 并行更强但成本和冲突上升。 | 复杂任务拆分能力下降。 |
| `subagentSummaryMaxTokens` | 返回父 Agent 的摘要上限 | tokens | `8_000` | 父 Agent 只需要结论、证据、文件和风险，不需要完整子 transcript。 | 子 Agent 完成后摘要超过 8k 就 microcompact 再返回。 | 父 Agent 获得更多细节，但上下文压力增大。 | 摘要更短，可能缺少证据链。 |
| `readOnlyFirstMilestone` | 第一版子 Agent 是否只读 | 布尔 | `true` | 第一版先让子 Agent 负责调查和验证，避免多个 Agent 并发写同一工作区。 | launchSubagent 时剔除 Edit/Write/危险 Bash，只保留读取类工具。 | 若关闭，子 Agent 可并行实现，但文件冲突和权限风险上升。 | 保持开启会让实现动作必须回到父 Agent。 |

## 详细流程

1. 读取 agent definition。
2. 校验 tools 是否是父 Agent 允许集合的子集。
3. 创建 sidechain transcript。
4. 复制 compact summary、最新用户意图、必要文件列表。
5. 运行子 Agent queryLoop。
6. 完成后用 summary prompt 生成摘要。
7. 父 Agent 只接收摘要和引用路径，不接收完整 transcript。

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| agent_not_found | 返回 unknown_agent_type。 |
| tool_not_allowed | 拒绝启动。 |
| spawn_limit_exceeded | 返回 budget_exceeded。 |
| subagent_timeout | 停止子 Agent，返回 partial summary。 |
| summary_too_large | microcompact 到 8_000 tokens。 |

## 提示词模板

### 子 Agent 总结提示词

~~~text
你是子 Agent 总结器。请把子 Agent 的运行结果压缩给父 Agent。
必须包含：任务、使用过的工具、读过或改过的文件、确定事实、失败、风险、不确定项、建议下一步。
禁止包含完整聊天历史。禁止把猜测写成事实。最多 8_000 tokens。
~~~

## 可实现伪代码

~~~ts
async function launchSubagent(req: SubagentLaunchRequest): Promise<SubagentResult> {
  const def = await loadAgentDefinition(req.agentType)
  if (!isToolSubset(def.tools, req.allowedTools)) throw new Error("tool_not_allowed")
  await enforceSpawnBudget(req.parentSessionId, 3)
  const child = await createSidechain(req.parentSessionId, def)
  const result = await queryLoop({ sessionId: child.sessionId, userMessage: makeUserMessage(req.prompt), tools: def.tools, maxTurns: req.maxTurns ?? 20 })
  const summary = await summarizeSubagent(child.transcriptPath, 8_000)
  return { agentId: child.agentId, transcriptPath: child.transcriptPath, summary, filesTouched: result.filesTouched, status: result.ok ? "succeeded" : "partial" }
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 启动 Explore | agentType=explore，tools=Read/Grep/Glob | 创建 sidechain，maxTurns=20。 |
| 工具越权 | 子 Agent 请求 Write 但父只允许 Read | tool_not_allowed。 |
| 超过启动上限 | 同一父 turn 已启动 3 个 | spawn_limit_exceeded。 |
| 摘要过大 | 子 Agent transcript 摘要 20_000 tokens | 压缩到 8_000 tokens。 |
| 子 Agent 失败 | 子 Agent timeout | 返回 partial summary，父 Agent 可继续。 |

## 验收标准

- 有具体默认值。
- 有结构化错误码。
- 有可执行伪代码。
- 测试覆盖正常路径、失败路径和边界值。
