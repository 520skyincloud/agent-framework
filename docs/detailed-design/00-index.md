# 详细设计索引

## 设计目标

本目录是智能体框架的工程实现手册。它不是学习笔记，而是把单 Agent 和多 Agent 软件需要的运行时、上下文、工具、权限、持久化、评估和部署全部拆成可实现专题。

推荐阅读顺序：01 主循环，02 消息，03 上下文，04 模型，05 Prompt，06 工具协议，07 工具编排，08 权限，09 文件，10 持久化，11 任务，12 多 Agent，13 MCP，14 记忆，15 事件，16 回放，17 预算，18 部署。

## 非目标

- 不解释大模型基础概念。
- 不绑定单一 provider。
- 不把所有细节塞回一篇长文。
- 不写中英文双语正文；正文以中文为主，代码标识保留英文。

## 核心规则

- 每个专题都有状态机、数据结构、默认值、流程、失败处理、伪代码、测试和验收。
- 所有影响下一轮模型行为的状态必须能持久化、恢复、回放。
- 所有副作用必须经过权限和预算。
- 所有自动恢复必须有次数上限。

## 状态机

~~~mermaid
stateDiagram-v2
  [*] --> ReadIndex
  ReadIndex --> SingleAgent
  SingleAgent --> Reliability
  Reliability --> MultiAgent
  MultiAgent --> Acceptance
  Acceptance --> [*]
~~~

## 数据结构

~~~ts
type DetailedDesignChapter = {
  order: number
  path: string
  title: string
  mustReadBeforeImplementation: boolean
  acceptance: string[]
}
~~~

## 默认值

| 参数名 | 中文含义 | 单位 | 默认值 | 为什么是这个值 | 触发行为 | 调大后果 | 调小后果 |
|---|---|---|---:|---|---|---|---|
| `detailedDesignFiles` | 详细设计文件总数，含索引和 18 个专题 | 个 | `19` | 19 个文件覆盖从 runtime 到部署的完整实现面，避免把多个独立主题塞进一个长文档。 | 文档检查时统计该目录，数量不是 19 就说明专题缺失或被误删。 | 增加专题会让维护更细，但阅读路线更长。 | 减少专题会让职责混在一起，实现者更难定位。 |
| `firstMilestoneDays` | 第一版可靠单 Agent 的建议实现周期 | 天 | `7` | 7 天只够做主循环、工具、权限、上下文和恢复闭环，不把多 Agent 当第一里程碑。 | 制定 MVP 排期时按该值切掉多 Agent、远程部署和高级记忆。 | 周期更长可以加入更多能力，但容易拖成大而不稳的原型。 | 周期更短会压缩测试和恢复设计。 |
| `firstMilestoneTools` | 第一版必须支持的核心工具数量 | 个 | `6` | Read、Grep、Glob、Bash、Edit、Write 是编码 Agent 可闭环的最小工具集。 | 第一版工具注册少于 6 个时，不应标记为可用框架。 | 加更多工具能覆盖更多场景，但权限和协议复杂度上升。 | 少于 6 个会缺少读取、搜索、执行或编辑能力。 |
| `minimumReplayCases` | 第一版最低回放用例数量 | 个 | `12` | 12 个用例能覆盖主循环、工具、权限、上下文、恢复和失败分支。 | CI 或 docs check 至少确认这些 replay case 存在。 | 覆盖更充分，但维护 fixture 成本更高。 | 回归风险上升，框架改动更难验证。 |

## 详细流程

1. 先实现 01、02、06、07，得到能跑通的最小主循环。
2. 加 03、04、05，解决上下文、模型路由和 Prompt 组装。
3. 加 08、09、10，解决副作用安全、文件编辑和恢复。
4. 加 11、12、13、14，扩展任务、多 Agent、MCP、记忆。
5. 加 15、16、17、18，完成生产可观测性、回放、预算和部署。

## 失败处理

| 失败 | 处理 |
|---|---|
| 专题缺默认值 | 阻塞合并，补齐具体数字。 |
| 专题缺伪代码 | 阻塞合并，补齐输入、输出、错误。 |
| 章节互相冲突 | 以更靠近运行时的章节为准，并在索引记录。 |

## 提示词模板

~~~text
你是智能体框架设计审查员。请检查章节是否包含设计目标、非目标、核心规则、状态机、数据结构、默认值、详细流程、失败处理、提示词模板、伪代码、测试用例和验收标准。只输出缺失项和不明确项。
~~~

## 可实现伪代码

~~~ts
function chooseChapter(topic: string): string[] {
  if (topic.includes("上下文")) return ["03-context-budget-and-compaction.md", "04-model-routing-and-downgrade.md"]
  if (topic.includes("工具")) return ["06-tool-protocol.md", "07-tool-execution-orchestration.md"]
  if (topic.includes("多 Agent")) return ["12-multi-agent-orchestration.md", "14-memory-system.md"]
  return ["01-runtime-loop.md", "02-message-and-transcript.md"]
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 查询上下文 | 上下文超过 20k 怎么办 | 返回 03 和 04。 |
| 查询工具 | 工具并发怎么排 | 返回 06 和 07。 |
| 查询多 Agent | 子 Agent 怎么合并 | 返回 12 和 14。 |

## 验收标准

- 本目录有 19 个 Markdown 文件。
- 每个文件都有固定结构。
- 每个文件都有伪代码。
- 03 和 04 包含明确阈值、恢复分支和测试。
