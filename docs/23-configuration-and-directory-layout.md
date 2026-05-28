# 配置系统与目录结构

## 本章目标

定义配置优先级、项目目录、用户目录和运行时目录。

完成本章后，读者应该知道这一层为什么存在、如何实现最小版本、哪些默认值不能随意改、以及如何验收。

## 核心概念

- 配置系统必须可解释。
- 本章能力必须有清晰的输入、输出、失败语义和测试边界。
- 任何影响模型下一步行为的状态，都必须能被记录、恢复或回放。

## 架构位置

本章位于 `config` 层。它不是孤立模块，而是和主循环、工具系统、上下文管理、持久化、权限或 SDK 事件流共同工作。实现时要明确本层是否拥有状态，是否会产生副作用，是否会改变下一轮模型输入。

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

建议接口命名保持直接，例如 `configConfig`、`configState`、`configEvent`、`configResult`。如果这些类型变得过大，优先拆分所有权，而不是把所有字段塞进一个全局对象。

## 默认值与关键数字

| 参数名 | 中文含义 | 单位 | 默认值 | 为什么是这个值 | 触发行为 | 调大后果 | 调小后果 |
|---|---|---|---:|---|---|---|---|
| `configSource` | 配置文件入口 | 路径 | `.agent/config.json` | 框架需要一个固定入口来读取项目级默认值，避免每个模块自己猜配置位置。 | 启动或加载项目时读取；缺失时使用内置默认值并提示初始化。 | 多配置入口会增加排查成本。 | 过少会导致项目级覆盖能力不足。 |
| `owner` | 模块所有权标记 | 文本 | `explicit module` | 要求实现者明确哪个模块拥有状态，防止多个模块同时修改同一份状态。 | 设计或实现模块边界时使用。 | 所有权过细会增加跨模块协调。 | 所有权过粗会形成全局对象。 |
| `testMode` | 测试模式要求 | 文本 | `replay case required` | 要求该模块至少能被 replay case 覆盖，避免只靠人工验证。 | 写测试和回放 fixture 时使用。 | 测试要求过高会拖慢原型。 | 测试要求过低会让重构风险变大。 |

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

本章的重点是把 `config` 层变成可实现、可测试、可恢复的工程边界。只要边界清楚，后续实现者就不需要靠猜。

## 参考蓝图细节

以下内容保留原始架构蓝图中的细节、表格和代码片段，供实现时逐项对照。

```text
CLI 参数 > 环境变量 > 项目配置 > 用户配置 > 组织配置 > 内置默认值
```

```text
~/.agent/
  config.json
  permissions.json
  sessions/
    2026-05-24/
      session_<id>/
        transcript.jsonl
        metadata.json
        tool-results/
          toolu_<id>.txt
        tasks/
          task_<id>.json
          task_<id>.out
        agents/
          agent_<id>/
            transcript.jsonl
            metadata.json
        file-history/
          <hash>.patch
        media/
          image_<id>.png
```

```ts
type AgentRuntimeConfig = {
  model: ModelRoute
  context: ContextBudgetConfig
  tools: {
    maxConcurrency: number
    resultPersistChars: number
    perMessageResultChars: number
  }
  bash: {
    defaultTimeoutMs: number
    maxTimeoutMs: number
    progressAfterMs: number
    autoBackgroundAfterMs: number
  }
  permissions: {
    mode: PermissionMode
    allow: string[]
    deny: string[]
  }
  storage: {
    rootDir: string
    maxSessionBytes: number
    maxToolOutputBytes: number
  }
  agents: {
    defaultMaxTurns: number
    forkMaxTurns: number
    maxConcurrentSubagents: number
  }
}
```
