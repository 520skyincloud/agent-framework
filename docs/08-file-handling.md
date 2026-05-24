> 中文标题：文件处理
>
> English title: File Handling
>
> 中文导读：本章说明 Read/Edit/Search 工具的具体限制，包括读取 token 上限、文件大小上限、编辑前必须读取、搜索结果限制和 VCS 排除规则。
>
> English guide: This chapter explains concrete Read/Edit/Search limits: read token caps, file size caps, read-before-edit, search limits, and VCS exclusions.

# 8. File Handling

## 8.1 Read Tool Defaults

| Setting | Value |
|---|---:|
| Default text output token limit | `25_000 tokens` |
| File size pre-read cap | `0.25 MB` / about `256 KB` |
| Default prompt line guidance | `2_000 lines` |
| File read cache entries | `100` |
| File read cache memory | `25 MB` |

Read should support:

- `file_path`,
- `offset`,
- `limit`,
- PDF `pages`,
- images,
- notebooks.

Do not persist Read output as a tool result file by default. That creates a loop where the model reads a persisted file that came from Read.

## 8.2 Edit Tool Rules

Recommended:

| Setting | Value |
|---|---:|
| Max editable file size | `1 GiB` |
| Require prior read for edits | yes |
| Preserve line endings | yes |
| Track file history before edit | yes |
| Validate exact string replacement | yes |

Edit must fail if:

- file was not read,
- file changed since last read,
- old string is not unique,
- path is outside allowed workspace,
- file is too large.

## 8.3 Search Tool Defaults

| Tool | Limit |
|---|---:|
| Grep default `head_limit` | `250` |
| Grep unlimited escape hatch | `head_limit=0` |
| Glob default max results | `100` |
| Grep result persistence | `20_000 chars` |
| Glob result persistence | `100_000 chars` |

Always exclude VCS directories:

- `.git`,
- `.svn`,
- `.hg`,
- `.bzr`,
- `.jj`,
- `.sl`.
