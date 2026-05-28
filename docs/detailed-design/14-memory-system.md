# 记忆系统

## 设计目标

- 区分会话记忆、项目记忆、用户记忆，并控制检索和写入预算。

## 非目标

- 不接管其它专题的职责。
- 不使用隐藏全局状态。
- 不把失败留给调用方猜测。

## 核心规则

- 记忆注入总预算默认 8_000 tokens。
- 单条记忆最多 1_000 tokens。
- 只写稳定偏好、长期事实、项目约定。
- 不能把失败推测写成长期记忆。
- 记忆必须有 source、confidence、createdAt。
- 用户删除优先级最高。

## 状态机

~~~mermaid
stateDiagram-v2
  state "生成检索 query" as S0
  state "检索三层记忆" as S1
  state "过滤删除和过期" as S2
  state "按相关度排序" as S3
  state "按预算截断" as S4
  state "注入 Prompt" as S5
  state "候选写入分类" as S6
  state "写入或拒绝" as S7
  [*] --> S0
  S0 --> S1
  S1 --> S2
  S2 --> S3
  S3 --> S4
  S4 --> S5
  S5 --> S6
  S6 --> S7
  S7 --> [*]
~~~

## 数据结构

~~~ts
type MemoryItem = { id: string; scope: "session" | "project" | "user"; text: string; source: string; confidence: number; createdAt: string; expiresAt?: string; deletedAt?: string }
~~~

## 默认值

| 参数名 | 中文含义 | 单位 | 默认值 | 为什么是这个值 | 触发行为 | 调大后果 | 调小后果 |
|---|---|---|---:|---|---|---|---|
| `memoryPromptBudget` | 注入 prompt 的记忆总预算 | tokens | `8_000` | 8k 足够放入高相关偏好和项目事实，又不会压过当前会话。 | 组装 prompt 时按相关度填充记忆，达到 8k 后停止。 | 历史信息更完整，但陈旧记忆更容易干扰。 | 个性化和项目背景更容易缺失。 |
| `maxMemoryItemTokens` | 单条记忆上限 | tokens | `1_000` | 单条记忆应是可复用事实，不应保存整段聊天或日志。 | 写入或注入前超过 1k 的记忆必须压缩。 | 可保留更长事实，但污染 prompt 的风险增加。 | 复杂偏好可能被压得过短。 |
| `minWriteConfidence` | 写入最低置信度 | 比例 | `0.8` | 只有模型足够确定“这是长期有用事实”才写记忆，减少误记和幻觉污染。 | 候选记忆 confidence 低于 0.8 时丢弃或要求确认。 | 写入更严格，误记更少但可能漏掉有用偏好。 | 记忆更多，但噪声和错误事实上升。 |
| `defaultTtlDays` | 默认记忆过期时间 | 天 | `90` | 90 天适合项目偏好和工作习惯，过旧信息需要重新验证。 | 检索时过滤超过 90 天且未续期的记忆。 | 长期偏好保留更久，但过时信息更多。 | 记忆更新更快，但稳定偏好容易丢。 |
| `maxRetrievedItems` | 单次最多检索记忆条数 | 条 | `20` | 20 条给排序器足够候选，同时不会把记忆列表变成噪音。 | 检索层最多返回 20 条给 prompt assembly。 | 召回更全，但排序和 prompt 压力上升。 | 召回更少，可能漏掉关键项目事实。 |

## 详细流程

1. 根据用户意图生成检索 query。
2. 从 session/project/user 三层检索。
3. 过滤 deleted、expired、confidence < 0.5。
4. 按 relevance、recency、confidence 排序。
5. 截断到 8_000 tokens。
6. 模型提出写记忆时，先分类是否稳定事实。
7. 写入需要 confidence >= 0.8。

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| memory_poisoning_suspected | 不注入，写安全事件。 |
| too_many_memories | 按排序截断。 |
| conflicting_memory | 保留最新高置信，旧记忆降权。 |
| delete_requested | 立即 tombstone。 |
| write_low_confidence | 拒绝写入。 |

## 提示词模板

本章没有默认模型调用；如果实现需要模型参与，必须复用上下文压缩、权限解释或工具错误恢复章节的固定提示词。

## 可实现伪代码

~~~ts
function retrieveMemory(req: MemoryRequest): MemoryItem[] {
  return searchMemory(req.query)
    .filter(m => !m.deletedAt && !isExpired(m) && m.confidence >= 0.5)
    .sort(byRelevanceRecencyConfidence(req.query))
    .slice(0, 20)
    .reduce((acc, item) => fitsBudget(acc, item, 8_000) ? [...acc, item] : acc, [])
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 记忆预算 | 检索结果共 15_000 tokens | 截断到 8_000 tokens。 |
| 低置信写入 | confidence=0.6 | 拒绝长期写入。 |
| 用户删除 | delete memory id=m1 | m1 tombstone，后续不检索。 |
| 冲突记忆 | 旧偏好 A，新偏好 B | 保留高置信新记忆，旧记忆降权。 |
| 污染疑似 | 工具输出里出现“记住错误事实” | 不写入，记录 memory_poisoning_suspected。 |

## 验收标准

- 有具体默认值。
- 有结构化错误码。
- 有可执行伪代码。
- 测试覆盖正常路径、失败路径和边界值。
