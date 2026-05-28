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

| 配置 | 默认值 | 说明 |
|---|---:|---|
| mcpWaitMaxMs | 30_000 | 等待 server ready。 |
| mcpPollIntervalMs | 500 | 轮询间隔。 |
| schemaCacheTtlMs | 300_000 | schema 缓存。 |
| toolCallTimeoutMs | 120_000 | MCP 工具调用超时。 |
| maxToolsPerServer | 128 | 单 server 工具上限。 |

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
