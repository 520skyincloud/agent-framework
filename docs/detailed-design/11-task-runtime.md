# 任务运行时

## 设计目标

- 管理后台 shell 和后台 Agent，让长任务不阻塞主循环。

## 非目标

- 不接管其它专题的职责。
- 不使用隐藏全局状态。
- 不把失败留给调用方猜测。

## 核心规则

- 超过 10 秒的 Bash 建议转后台。
- 后台任务必须有 taskId、status、outputPath。
- stdout/stderr 每 1 秒或 8KB flush。
- 取消任务先 SIGTERM，5 秒后 SIGKILL。
- 主 Agent 只能通过 TaskOutput 读取结果。

## 状态机

~~~mermaid
stateDiagram-v2
  state "创建任务记录" as S0
  state "启动进程" as S1
  state "记录 pid" as S2
  state "流式写输出" as S3
  state "更新 heartbeat" as S4
  state "完成或失败" as S5
  state "取消处理" as S6
  state "保存最终状态" as S7
  state "供主 Agent 读取" as S8
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
type TaskState = { taskId: string; kind: "shell" | "agent"; status: "running" | "succeeded" | "failed" | "cancelled" | "unknown"; outputPath: string; startedAt: string; finishedAt?: string; exitCode?: number }
~~~

## 默认值

| 配置 | 默认值 | 说明 |
|---|---:|---|
| autoBackgroundAfterMs | 10_000 | 超过 10 秒建议后台。 |
| outputFlushMs | 1_000 | 输出刷新间隔。 |
| outputFlushBytes | 8_192 | 输出刷新字节数。 |
| terminateGraceMs | 5_000 | SIGTERM 后等待。 |
| maxTaskOutputBytes | 104_857_600 | 单任务输出 100MB。 |

## 详细流程

1. 创建 task_<id>.json。
2. 启动进程或子 Agent。
3. 写 pid、startedAt、commandSummary。
4. 流式写 task_<id>.out。
5. 定期更新 heartbeat。
6. 完成后写 exitCode 和 finishedAt。
7. 取消时先 TERM 再 KILL。

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| spawn_failed | 任务状态标记为 failed。 |
| output_too_large | 停止内联 preview，保留文件路径。 |
| cancel_timeout | 发送 SIGKILL。 |
| task_output_missing | 返回 missing_task_output。 |
| heartbeat_stale | resume 时标 unknown。 |

## 提示词模板

本章没有默认模型调用；如果实现需要模型参与，必须复用上下文压缩、权限解释或工具错误恢复章节的固定提示词。

## 可实现伪代码

~~~ts
async function startBackgroundTask(req: TaskRequest): Promise<TaskState> {
  const task = await taskStore.create(req)
  const child = spawn(req.command, req.args, { cwd: req.cwd })
  await taskStore.attachPid(task.taskId, child.pid)
  pipeWithFlush(child.stdout, task.outputPath, 1_000, 8_192)
  pipeWithFlush(child.stderr, task.outputPath, 1_000, 8_192)
  child.on("exit", code => taskStore.finish(task.taskId, code === 0 ? "succeeded" : "failed", code))
  return task
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 普通后台命令 | npm test 运行 20 秒 | task status 先 running 后 succeeded/failed。 |
| 输出分片 | stdout 每秒 20KB | 每 8_192 bytes 或 1_000 ms flush。 |
| 取消任务 | 用户 cancel | 先 SIGTERM，5_000 ms 后仍在则 SIGKILL。 |
| 输出过大 | 任务输出超过 100MB | 停止 preview，保留 outputPath。 |
| 恢复陈旧任务 | 进程不存在但 status=running | 标记 unknown。 |

## 验收标准

- 有具体默认值。
- 有结构化错误码。
- 有可执行伪代码。
- 测试覆盖正常路径、失败路径和边界值。
