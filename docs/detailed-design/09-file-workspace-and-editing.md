# 文件工作区与编辑

## 设计目标

- 把 Read、Grep、Glob、Edit、Write 做成可恢复、可审计、不会误覆盖用户改动的文件系统能力。

## 非目标

- 不接管其它专题的职责。
- 不使用隐藏全局状态。
- 不把失败留给调用方猜测。

## 核心规则

- Edit 前必须 Read。
- Edit 时必须校验 readHash。
- Write 新文件必须确认父目录在 workspace。
- 单次 Read 默认最多 25_000 tokens。
- 文件大于 262_144 bytes 默认拒绝全文读取。
- 每次写入保存 patch 到 file-history。

## 状态机

~~~mermaid
stateDiagram-v2
  state "Read 记录文件状态" as S0
  state "生成编辑请求" as S1
  state "检查路径权限" as S2
  state "校验 expectedSha256" as S3
  state "匹配 oldString" as S4
  state "应用修改到内存" as S5
  state "原子写入" as S6
  state "保存 patch" as S7
  state "更新文件缓存" as S8
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
type FileReadState = { path: string; sha256: string; mtimeMs: number; sizeBytes: number; readAt: string }
type EditRequest = { path: string; oldString: string; newString: string; expectedSha256: string }
~~~

## 默认值

| 配置 | 默认值 | 说明 |
|---|---:|---|
| readMaxOutputTokens | 25_000 | Read 输出 token 上限。 |
| readMaxSizeBytes | 262_144 | 默认全文读取字节上限。 |
| grepHeadLimit | 250 | Grep 默认最多返回 250 行。 |
| globMaxResults | 100 | Glob 默认最多 100 个路径。 |
| staleEditProtection | true | 编辑前校验 hash。 |

## 详细流程

1. Read 记录 path、mtime、size、sha256。
2. Edit 请求必须带 oldString 或 patch。
3. 编辑前重新 stat 和 hash。
4. hash 不一致返回 stale_edit。
5. 应用 patch 到内存。
6. 写入临时文件后 rename。
7. 保存反向 patch 和正向 patch。
8. 更新 read cache。

## 失败处理

| 错误码或失败 | 处理 |
|---|---|
| file_not_read | Edit 返回 file_not_read。 |
| stale_edit | 提示重新 Read。 |
| old_string_not_found | 不写文件，返回可恢复错误。 |
| multiple_matches | 要求更具体 oldString。 |
| outside_workspace | 返回 permission_denied。 |

## 提示词模板

本章没有默认模型调用；如果实现需要模型参与，必须复用上下文压缩、权限解释或工具错误恢复章节的固定提示词。

## 可实现伪代码

~~~ts
async function applyWorkspaceEdit(req: EditRequest, state: FileStateCache): Promise<EditResult> {
  const known = state.get(req.path)
  if (!known) return { ok: false, code: "file_not_read" }
  const current = await hashFile(req.path)
  if (current !== req.expectedSha256) return { ok: false, code: "stale_edit" }
  const text = await fs.readFile(req.path, "utf8")
  const count = countOccurrences(text, req.oldString)
  if (count === 0) return { ok: false, code: "old_string_not_found" }
  if (count > 1) return { ok: false, code: "multiple_matches" }
  const next = text.replace(req.oldString, req.newString)
  await atomicWrite(req.path, next)
  await savePatch(req.path, text, next)
  return { ok: true, newSha256: sha256(next) }
}
~~~

## 测试用例

| 用例 | 输入 | 期望 |
|---|---|---|
| 未读先改 | Edit(a.ts) 但 cache 无记录 | file_not_read。 |
| 用户同时修改 | expectedSha256 与当前文件不同 | stale_edit。 |
| oldString 不存在 | oldString=foo 当前文件无 foo | old_string_not_found。 |
| oldString 多处匹配 | 同一片段出现 2 次 | multiple_matches。 |
| 大文件读取 | 文件 400_000 bytes | 拒绝全文读取，要求 range 或 grep。 |

## 验收标准

- 有具体默认值。
- 有结构化错误码。
- 有可执行伪代码。
- 测试覆盖正常路径、失败路径和边界值。
