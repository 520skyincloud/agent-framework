# 多 Agent 架构

## 本章目标

定义子 Agent 类型、隔离、通信、父子协议、权限冒泡和拓扑。

完成本章后，读者应该知道这一层为什么存在、如何实现最小版本、哪些默认值不能随意改、以及如何验收。

## 核心概念

- 多 Agent 难点在隔离和合并。
- 本章能力必须有清晰的输入、输出、失败语义和测试边界。
- 任何影响模型下一步行为的状态，都必须能被记录、恢复或回放。

## 架构位置

本章位于 `multiagent` 层。它不是孤立模块，而是和主循环、工具系统、上下文管理、持久化、权限或 SDK 事件流共同工作。实现时要明确本层是否拥有状态，是否会产生副作用，是否会改变下一轮模型输入。

## 具体设计

最小设计应包含四个部分：输入对象、输出对象、错误对象和持久化记录。输入对象用于阻止隐式全局依赖；输出对象用于让 UI、SDK 和 replay 共用同一结果；错误对象用于让模型或用户知道下一步怎么恢复；持久化记录用于崩溃后继续运行。

成熟设计还应该补充可观测性和预算控制。只要本章能力可能变慢、变贵、失败或产生副作用，就必须发出事件并记录关键 ID。

## 接口与数据结构

| 边界 | 说明 |
|---|---|
| 输入 | 调用方必须显式传入的状态和参数。 |
| 输出 | 成功时返回的数据、事件或状态变更。 |
| 错误 | 失败时返回给用户、模型或调用方的结构化错误。 |
| 持久化 | 必须写入 transcript、metadata 或输出文件的内容。 |
| 回放 | 回放测试需要记录和模拟的输入输出。 |

建议接口命名保持直接，例如 `multiagentConfig`、`multiagentState`、`multiagentEvent`、`multiagentResult`。如果这些类型变得过大，优先拆分所有权，而不是把所有字段塞进一个全局对象。

## 默认值与关键数字

| 参数名 | 中文含义 | 单位 | 默认值 | 为什么是这个值 | 触发行为 | 调大后果 | 调小后果 |
|---|---|---|---:|---|---|---|---|
| `defaultMaxTurns` | 普通子 Agent 最大轮数 | 轮 | `20` | 普通子 Agent 应完成一个明确子问题，20 轮足够调查、读取、验证和总结。 | 子 Agent 达到 20 轮时停止，并把 partial summary 返回父 Agent。 | 子 Agent 更容易独立完成复杂调查，但预算和跑偏风险上升。 | 父 Agent 需要更频繁拆分任务。 |
| `forkMaxTurns` | fork 型子 Agent 最大轮数 | 轮 | `200` | fork Agent 用于长调查或后台验证，200 轮给足空间但仍有硬上限。 | fork 子 Agent 达到 200 轮后停止，输出摘要和未完成项。 | 长调查更完整，但成本、延迟和上下文漂移上升。 | 深度验证更容易中断。 |
| `mcpWaitMaxMs` | MCP 连接等待上限 | 毫秒 | `30_000` | 30 秒覆盖本地 MCP server 启动和握手，避免坏连接拖住子 Agent。 | 等待 MCP ready 超过 30 秒时标记 server unavailable。 | 慢服务更容易连上，但失败反馈更晚。 | 慢启动服务更容易被误判不可用。 |
| `mcpPollIntervalMs` | MCP ready 轮询间隔 | 毫秒 | `500` | 500ms 能较快发现 MCP ready，又不会高频占用 CPU。 | 等待 MCP 时每 500ms 查询一次 ready 状态。 | CPU 更省，但 ready 后开始执行更慢。 | 响应更快，但轮询噪音更高。 |

如果本章没有专属数字，就使用 `.agent/config.json` 中的全局默认值；新增参数时也必须补齐“含义、单位、默认值原因、触发行为、调大后果、调小后果”。

## 实现步骤

1. 先实现最小闭环，再添加高级能力。
2. 定义输入、输出、错误和持久化边界。
3. 把默认值集中到配置或常量模块。
4. 为正常路径、失败路径和边界值写测试。
5. 把影响下一轮模型行为的状态写入 transcript、metadata 或 replay fixture。

## 测试与验收

- 正常路径必须产出符合接口的结果。
- 失败路径必须返回结构化错误，而不是静默失败。
- 达到默认限制时必须触发文档规定的行为。
- 恢复或回放时结果必须可解释。
- 相关验收标准必须能被自动化测试验证。

## 常见错误

- 只写概念，没有写输入输出和验收。
- 把默认数字散落在多个实现文件。
- 失败时直接 throw，导致主循环无法恢复。
- 没有 replay case，后续重构容易破坏行为。

## 本章总结

本章的重点是把 `multiagent` 层变成可实现、可测试、可恢复的工程边界。只要边界清楚，后续实现者就不需要靠猜。

## 参考蓝图细节

以下内容保留原始架构蓝图中的细节、表格和代码片段，供实现时逐项对照。

```yaml
name: Explore
description: 只调查代码，不编辑文件
tools:
  - Read
  - Grep
  - Glob
  - Bash
permissionMode: plan
maxTurns: 20
model: sonnet
background: false
mcpServers:
  - github
skills:
  - code-review
```

```ts
type SubagentLaunchRequest = {
  parentAgentId?: string
  prompt: string
  agentType: string
  model?: string
  maxTurns: number
  background: boolean
  cwd: string
  allowedTools: string[]
  permissionMode: PermissionMode
  forkContextMessages?: Message[]
  inheritedFiles?: string[]
}

type SubagentLaunchResult =
  | { mode: "sync"; agentId: string; finalText: string; transcriptPath: string }
  | { mode: "async"; agentId: string; taskId: string; outputPath: string }
```

```text
主 Agent
  - 保留用户会话
  - 拥有权限决策权
  - 决定任务如何拆分
  - 合并最终答案

探索 Agent
  - 只读
  - maxTurns = 20
  - 返回文件地图和调查结论

验证 Agent
  - 可读文件，也可运行 bash
  - background = true
  - 验证测试、构建和浏览器结果

TaskOutput
  - 允许主 Agent 读取异步验证结果
```
