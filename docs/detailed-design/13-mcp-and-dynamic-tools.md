# MCP 与动态工具

## 设计目标

- 连接 MCP server，动态发现工具，并安全加入工具注册表。

## 非目标

- 不接管其它专题的职责。
- 不使用隐藏全局状态。
- 不把失败留给调用方猜测。

## 核心规则

- MCP server 启动等待最多 30_000 ms。
- 轮询间隔 500 ms。
- 工具命名格式 mcp__server__tool。
- schema 缓存 300 秒。
- 单个 MCP 失败不能拖垮主循环。
- 动态工具默认需要权限确认。

## 状态机

~~~mermaid
stateDiagram-v2
  state "读取 MCP 配置" as S0
  state "连接 server" as S1
  state "等待 ready" as S2
  state "拉取工具 schema" as S3
  state "加命名空间" as S4
  state "缓存 schema" as S5
  state "注册工具" as S6
  state "调用时转发" as S7
  state "隔离失败 server" as S8
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
type McpToolRef = { serverName: string; toolName: string; runtimeName: string; inputSchema: unknown; cachedAt: string }
~~~

## 默认值

| 参数名 | 中文含义 | 单位 | 默认值 | 为什么是这个值 | 触发行为 | 调大后果 | 调小后果 |
|---|---|---|---:|---|---|---|---|
| `mcpWaitMaxMs` | MCP 连接等待上限 | 毫秒 | `30_000` | 30 秒能覆盖本地服务启动，同时避免一直等坏连接。 | 连接 MCP server 时使用。 | 慢服务更容易连上。 | 问题服务阻塞更久。 |
| `mcpPollIntervalMs` | MCP 轮询间隔 | 毫秒 | `500` | 500ms 兼顾响应速度和 CPU 开销。 | 等待 MCP ready 时轮询。 | CPU 更省但 ready 反馈慢。 | 响应更快但轮询更频繁。 |
| `schemaCacheTtlMs` | schema 缓存 | 毫秒 | `300_000` | 这个值按 token 预算设置，用来把关键上下文放进模型输入，同时避免某一类内容挤掉最新用户意图。 | 组装 prompt 或计算 token 预算时使用。 | 该类内容可保留更多，但会挤压其它上下文。 | prompt 更紧凑，但可能丢失必要背景。 |
| `toolCallTimeoutMs` | MCP 工具调用超时 | 毫秒 | `120_000` | 这个时间值用于区分正常等待和疑似卡死，第一版优先保证任务不会无限挂起。 | 运行时间或等待时间超过该值时触发超时、刷新、心跳或清理。 | 更不容易误杀慢任务，但卡住时等待更久。 | 故障暴露更快，但慢任务更容易被误判。 |
| `maxToolsPerServer` | 单 server 工具上限 | 个 | `128` | 这个数量限制用于控制一次任务的执行规模，防止循环、并发或子任务无限扩张。 | 数量达到该值时停止、排队、截断或要求用户确认。 | 吞吐和覆盖面更高，但成本、冲突和排查难度上升。 | 系统更稳，但复杂任务更容易分多轮完成。 |

## 详细流程

1. 读取 mcpServers 配置。
2. 启动或连接 server。
3. 等待 ready，最多 30 秒。
4. 调用 listTools。
5. 为每个工具加命名空间。
6. 缓存 schema。
7. 注册到 ToolRegistry。
8. 调用时把 MCP 错误转 ToolResult。

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| server_timeout | 跳过该 server，记录 mcp_server_unavailable。 |
| schema_invalid | 跳过该工具。 |
| name_collision | 使用命名空间后仍冲突则拒绝。 |
| tool_call_timeout | 返回 timeout tool_result。 |
| server_crash | 隔离该 server。 |

## 提示词模板

本章没有默认模型调用；如果实现需要模型参与，必须复用上下文压缩、权限解释或工具错误恢复章节的固定提示词。

## 可实现伪代码

~~~ts
async function loadDynamicTools(config: McpConfig): Promise<ToolDefinition[]> {
  const tools: ToolDefinition[] = []
  for (const server of config.servers) {
    const client = await connectWithTimeout(server, 30_000).catch(() => null)
    if (!client) continue
    const schemas = await client.listTools()
    for (const schema of schemas.slice(0, 128)) tools.push(wrapMcpTool(server.name, schema))
  }
  return tools
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| server ready 慢 | 30_001 ms 后仍未 ready | server_timeout，跳过。 |
| 工具过多 | server 返回 200 个工具 | 只注册前 128 个并记录截断。 |
| schema 非法 | inputSchema 不是对象 | 跳过该工具。 |
| 调用超时 | MCP 工具超过 120_000 ms | 返回 timeout tool_result。 |
| server 崩溃 | 调用中连接断开 | 隔离 server，不影响其它工具。 |

## 验收标准

- 有具体默认值。
- 有结构化错误码。
- 有可执行伪代码。
- 测试覆盖正常路径、失败路径和边界值。
