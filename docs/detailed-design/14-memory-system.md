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

| 配置 | 默认值 | 说明 |
|---|---:|---|
| memoryPromptBudget | 8_000 | 注入 Prompt 的总预算。 |
| maxMemoryItemTokens | 1_000 | 单条记忆上限。 |
| minWriteConfidence | 0.8 | 写入最低置信度。 |
| defaultTtlDays | 90 | 默认过期时间。 |
| maxRetrievedItems | 20 | 最多检索 20 条。 |

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
