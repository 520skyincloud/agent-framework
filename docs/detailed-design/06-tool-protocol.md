# 工具协议

## 设计目标

- 定义每个工具必须实现的接口、权限、输出、错误和 provider 映射。

## 非目标

- 不接管其它专题的职责。
- 不使用隐藏全局状态。
- 不把失败留给调用方猜测。

## 核心规则

- 工具 name 全局唯一。
- inputSchema 必须可运行时校验。
- 副作用工具必须实现 isDestructive 或 isReadOnly。
- 工具返回值必须是 ToolResult，不允许直接 throw 给主循环。
- 输出超过 50_000 字符落盘。
- 工具错误也必须转成 tool_result 返回模型。

## 状态机

~~~mermaid
stateDiagram-v2
  state "接收 tool_use" as S0
  state "查找工具" as S1
  state "校验 inputSchema" as S2
  state "语义校验" as S3
  state "权限检查" as S4
  state "启动超时保护" as S5
  state "执行工具" as S6
  state "落盘大输出" as S7
  state "映射 tool_result" as S8
  state "返回主循环" as S9
  [*] --> S0
  S0 --> S1
  S1 --> S2
  S2 --> S3
  S3 --> S4
  S4 --> S5
  S5 --> S6
  S6 --> S7
  S7 --> S8
  S8 --> S9
  S9 --> [*]
~~~

## 数据结构

~~~ts
type ToolDefinition<I,O> = { name: string; inputSchema: Schema<I>; isReadOnly(input:I): boolean; checkPermissions(input:I,ctx:ToolContext): Promise<PermissionDecision>; call(input:I,ctx:ToolContext,onProgress:(p:ToolProgress)=>void): Promise<O>; mapResult(output:O,toolUseId:string): ToolResultMessage }
~~~

## 默认值

| 配置 | 默认值 | 说明 |
|---|---:|---|
| maxInlineResultChars | 50_000 | 工具结果内联上限。 |
| previewChars | 2_000 | 落盘结果预览字符数。 |
| defaultToolTimeoutMs | 120_000 | 普通工具超时。 |
| bashDefaultTimeoutMs | 120_000 | Bash 默认超时。 |
| bashMaxTimeoutMs | 600_000 | Bash 最大超时。 |
| progressAfterMs | 1_000 | 超过 1 秒发 progress。 |

## 详细流程

1. 查找 tool name。
2. 用 inputSchema 校验输入。
3. 调用 validateInput 做语义校验。
4. 调用 checkPermissions。
5. 启动 timeout watchdog。
6. 执行 call，期间发 progress。
7. 结果超过 50_000 字符先写文件。
8. 映射成 provider tool_result block。

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| unknown_tool | 合成错误 tool_result：工具不存在。 |
| invalid_input | 合成错误 tool_result：输入不符合 schema。 |
| permission_denied | 合成错误 tool_result：权限拒绝。 |
| timeout | 尝试取消工具，返回可恢复错误。 |
| result_too_large | 落盘并返回 preview。 |

## 提示词模板

### 工具错误恢复提示词

~~~text
你刚才的工具调用失败了。请根据 tool_result 中的错误码修正下一步。
规则：
1. invalid_input：重新生成符合 schema 的输入。
2. permission_denied：不要重复相同副作用请求，先向用户解释需要什么权限。
3. timeout：缩小命令范围，或改用后台任务。
4. output_persisted：读取 preview 和路径，不要要求把完整输出塞回上下文。
5. unknown_tool：改用可用工具，不要发明工具名。
~~~

## 可实现伪代码

~~~ts
async function runTool(req: ToolRunRequest): Promise<ToolRunResult> {
  const tool = registry.get(req.name)
  if (!tool) return errorToolResult(req, "unknown_tool")
  const parsed = tool.inputSchema.safeParse(req.input)
  if (!parsed.success) return errorToolResult(req, "invalid_input")
  const permission = await tool.checkPermissions(parsed.data, req.ctx)
  if (permission.decision !== "allow") return errorToolResult(req, permission.decision === "ask" ? "permission_required" : "permission_denied")
  const output = await withTimeout(tool.call(parsed.data, req.ctx, req.onProgress), req.timeoutMs)
  return persistAndMapToolOutput(tool, output, req.toolUseId, 50_000)
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 未知工具 | tool name=UnknownTool | 返回 unknown_tool 的错误 tool_result。 |
| 输入非法 | Read 缺 path | 返回 invalid_input，工具不执行。 |
| 权限拒绝 | Write(/etc/hosts) | 返回 permission_denied，写审计。 |
| 输出过大 | Bash 输出 75_000 字符 | 全文写 tool-results，tool_result 只有 2_000 字符 preview。 |
| 工具超时 | Bash 超过 120_000 ms | 取消进程，返回 timeout tool_result。 |

## 验收标准

- 有具体默认值。
- 有结构化错误码。
- 有可执行伪代码。
- 测试覆盖正常路径、失败路径和边界值。
