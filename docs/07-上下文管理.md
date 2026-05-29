# 上下文管理

## 本章目标

说明 token 预算、自动压缩、手动阻塞、工具结果瘦身和反模式。读者看完后应该能回答三个问题：当前 prompt 还能不能发给模型、什么时候必须压缩、压缩失败后什么时候必须停下来。

## 核心概念

- 上下文管理必须提前行动，不能等 provider 返回 prompt too long 才补救。
- 上下文窗口不是全部都能给 prompt 使用；必须给模型输出和压缩摘要预留空间。
- warning、auto compact、block 是三个不同层级：warning 只提醒，auto compact 会自动改写上下文，block 会禁止直接调用模型。
- 所有阈值必须能通过配置复现，不能散落在业务逻辑里。

## 架构位置

本章位于 `context` 层。它在模型调用前运行，读取模型窗口、输出上限、当前 prompt token 估算和压缩失败次数，然后返回明确动作：继续、提示、自动压缩、完整压缩或阻塞。

## 为什么要预留输出空间

如果模型上下文是 `200_000 tokens`，不能把 `200_000 tokens` 全部塞给 prompt。模型还需要空间输出回答、工具调用、压缩摘要或错误恢复说明。否则请求即使被 provider 接收，也可能因为没有输出空间而中途截断。

所以先计算有效窗口：

`effectiveWindow = contextWindow - min(modelMaxOutputTokens, 20_000)`

逐项解释：

| 名称 | 中文含义 | 单位 | 为什么存在 |
|---|---|---|---|
| `contextWindow` | 目标模型总上下文窗口 | tokens | 不同模型窗口不同，预算必须按目标模型算，不按历史模型算。 |
| `modelMaxOutputTokens` | 本次调用允许模型输出的最大 token 数 | tokens | 输出越大，prompt 可用空间越小。 |
| `20_000` | 输出/压缩摘要预留上限 | tokens | 大模型最多预留 20k，避免 prompt 占满上下文；小模型会使用更小上限。 |
| `effectiveWindow` | prompt 真正可用的最大窗口 | tokens | 后续 warning、compact、block 都基于这个值计算。 |

## 默认值与关键数字

| 参数名 | 中文含义 | 单位 | 默认值 | 为什么是这个值 | 触发行为 | 调大后果 | 调小后果 |
|---|---|---|---:|---|---|---|---|
| `contextWindowTokens` | 默认目标模型上下文窗口 | tokens | `200_000` | 200k 是第一版可靠编程 Agent 的保守基线，能容纳较长代码任务，又不假设所有模型都有 1M 窗口。 | 每次模型调用前参与预算计算。 | 可保留更多历史，但成本、延迟和压缩摘要复杂度上升。 | 更频繁触发压缩，长任务更容易阻塞。 |
| `largeContextWindowTokens` | 大上下文模型窗口 | tokens | `1_000_000` | 1M 用于长仓库、长 transcript 或多 Agent 总结，不作为所有模型默认假设。 | 路由到大上下文模型时使用。 | 能保留更多历史，但调用更贵更慢。 | 大会话更早需要压缩。 |
| `modelMaxOutputTokens` | 本次模型最大输出预算 | tokens | `20_000` | 20k 足够输出长摘要、计划或工具调用，同时不会让 prompt 空间过小。 | 计算 `summaryReserve`。 | 输出更完整，但 prompt 可用空间减少。 | 输出可能被截断，压缩摘要可能不完整。 |
| `summaryReserveTokens` | 输出/压缩摘要预留空间 | tokens | `20_000` | 给模型留出回答和 compact 结果空间，防止 prompt 顶满窗口。 | 从 `contextWindow` 中扣除。 | 压缩更早触发，但输出更安全。 | prompt 空间更大，但输出截断风险更高。 |
| `autoCompactBufferTokens` | 自动压缩缓冲 | tokens | `13_000` | 给 token 估算误差、工具 schema 和 provider 包装留余量。 | `promptTokens > autoCompactThreshold` 时触发自动压缩。 | 更早压缩，更安全但更频繁。 | 更晚压缩，prompt too long 风险上升。 |
| `manualReserveTokens` | 阻塞前最后保留空间 | tokens | `3_000` | 给错误说明、恢复消息和 provider 包装留下最后余量。 | `promptTokens > blockingLimit` 时禁止直接调用模型。 | 更早阻塞，用户更常看到清理提示。 | 更激进，但更容易被 provider 拒绝。 |
| `warningBufferTokens` | 提前警告缓冲 | tokens | `20_000` | 在真正压缩前提前提醒，给 UI 和用户留操作空间。 | `promptTokens > warningThreshold` 时发 warning。 | 提醒更早但可能打扰。 | 提醒更晚，用户更难提前处理。 |
| `maxConsecutiveAutoCompactFailures` | 自动压缩连续失败熔断次数 | 次 | `3` | 3 次能覆盖偶发模型失败，同时避免无限压缩循环。 | 第 3 次失败后 block。 | 恢复机会更多，但成本和等待时间上升。 | 更快失败，但偶发问题更容易暴露给用户。 |

## 公式解释

| 公式 | 含义 | 达到后系统做什么 |
|---|---|---|
| `effectiveWindow = contextWindow - min(modelMaxOutputTokens, 20_000)` | 计算 prompt 真正可用空间。 | 后续所有阈值都基于它计算。 |
| `autoCompactThreshold = effectiveWindow - 13_000` | 自动压缩阈值。 | prompt 超过它时，先 compact，再调用模型。 |
| `blockingLimit = effectiveWindow - 3_000` | 阻塞阈值。 | prompt 超过它时，禁止直接调用模型，必须 full compact 或要求用户确认。 |
| `warningThreshold = autoCompactThreshold - 20_000` | 提醒阈值。 | prompt 超过它但没到自动压缩时，只发 warning。 |
| `errorThreshold = warningThreshold` | 第一版错误提示阈值。 | 第一版 warning/error 共用阈值；UI 可用不同文案展示。 |

## 计算例子

### 200k 模型

| 步骤 | 计算 | 结果 | 解释 |
|---|---|---:|---|
| 总窗口 | `contextWindow` | `200_000` | 目标模型最多能接收 200k tokens。 |
| 输出预留 | `min(modelMaxOutputTokens, 20_000)` | `20_000` | 给模型回答或压缩摘要留空间。 |
| 有效窗口 | `200_000 - 20_000` | `180_000` | prompt 最多只能安全使用 180k。 |
| 自动压缩 | `180_000 - 13_000` | `167_000` | 超过 167k 就先 compact。 |
| 阻塞限制 | `180_000 - 3_000` | `177_000` | 超过 177k 不能直接调用模型。 |
| warning/error | `167_000 - 20_000` | `147_000` | 超过 147k 开始提醒。 |

200k 下的动作区间：

| promptTokens | 动作 | 原因 |
|---:|---|---|
| `<= 147_000` | continue | 还有足够余量。 |
| `147_001` 到 `167_000` | warning | 接近自动压缩，但还不用改写上下文。 |
| `167_001` 到 `177_000` | auto compact | 已进入压缩区，先压缩再调用模型。 |
| `> 177_000` | block/full compact | 直接调用模型风险太高，必须完整压缩。 |

### 1M 模型

| 步骤 | 计算 | 结果 | 解释 |
|---|---|---:|---|
| 总窗口 | `contextWindow` | `1_000_000` | 大上下文模型窗口。 |
| 输出预留 | `min(modelMaxOutputTokens, 20_000)` | `20_000` | 仍只预留 20k，不无限扩大输出空间。 |
| 有效窗口 | `1_000_000 - 20_000` | `980_000` | prompt 安全空间。 |
| 自动压缩 | `980_000 - 13_000` | `967_000` | 超过后自动 compact。 |
| 阻塞限制 | `980_000 - 3_000` | `977_000` | 超过后禁止直接调用模型。 |
| warning/error | `967_000 - 20_000` | `947_000` | 提前提醒。 |

### 20k 模型

小模型不能继续使用 20k 作为输出预留，否则 prompt 空间会接近 0。切到 20k 模型时，预留规则改为：`min(modelMaxOutputTokens, 4_000)`。

| 步骤 | 计算 | 结果 | 解释 |
|---|---|---:|---|
| 总窗口 | `contextWindow` | `20_000` | 目标小模型窗口。 |
| 输出预留 | `min(modelMaxOutputTokens, 4_000)` | `4_000` | 小模型最多预留 4k 输出。 |
| 有效窗口 | `20_000 - 4_000` | `16_000` | prompt 安全空间。 |
| 自动压缩 | `16_000 - 13_000` | `3_000` | 超过 3k 就应进入 downgrade compact。 |
| 阻塞限制 | `16_000 - 3_000` | `13_000` | 超过 13k 不能直接调用。 |

## 降级场景

从 1M 或 200k 会话切到 20k 模型时，不能保留完整历史。必须执行 downgrade compact，只保留：目标、当前状态、关键文件、未完成任务、工具结果摘要、风险。

如果 downgrade compact 后仍超过目标窗口：

| 次数 | 动作 | 保留内容 |
|---:|---|---|
| 第 1 次 | 更激进 compact | 只保留和当前目标直接相关的信息。 |
| 第 2 次 | task state + latest user intent | 只保留任务状态和最新用户意图。 |
| 第 3 次 | block | 要求用户确认清理历史或换回大上下文模型。 |

## 可实现伪代码

```ts
type ContextDecision = "continue" | "warn" | "auto_compact" | "full_compact" | "block"

function getContextDecision(inputTokens: number, cfg: ContextBudgetConfig): ContextDecision {
  const reserveCap = cfg.contextWindowTokens <= 20_000 ? 4_000 : 20_000
  const summaryReserve = Math.min(cfg.modelMaxOutputTokens, reserveCap)
  const effectiveWindow = cfg.contextWindowTokens - summaryReserve
  const autoCompactAt = effectiveWindow - cfg.autoCompactBufferTokens
  const blockAt = effectiveWindow - cfg.manualCompactReserveTokens
  const warnAt = autoCompactAt - cfg.warningBufferTokens

  if (cfg.consecutiveAutoCompactFailures >= 3) return "block"
  if (inputTokens > blockAt) return "full_compact"
  if (inputTokens > autoCompactAt) return "auto_compact"
  if (inputTokens > warnAt) return "warn"
  return "continue"
}
```

## 测试与验收

| 用例 | 输入 | 期望 |
|---|---|---|
| 200k 安全区 | `promptTokens=140_000` | continue。 |
| 200k warning | `promptTokens=150_000` | warn。 |
| 200k 自动压缩 | `promptTokens=170_000` | auto_compact。 |
| 200k 完整压缩 | `promptTokens=178_000` | full_compact，不直接调用模型。 |
| 1M 自动压缩 | `promptTokens=970_000` | auto_compact。 |
| 20k 降级 | 200k 会话切到 20k 模型 | downgrade compact。 |
| 压缩三次失败 | `consecutiveAutoCompactFailures=3` | block。 |

## 常见错误

- 把 `contextWindow` 全部给 prompt 使用，导致模型没有输出空间。
- warning、auto compact、block 用同一个阈值，导致行为不可解释。
- 从 1M 切 20k 时继续保留完整历史。
- compact 失败后无限重试。
- 只写公式，不解释每个数字的用途。
