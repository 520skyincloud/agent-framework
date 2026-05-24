> 中文标题：实施检查清单
>
> English title: Implementation Checklist
>
> 中文导读：本章提供上线前检查清单，覆盖核心循环、工具、权限、上下文、多 Agent 和持久化。
>
> English guide: This chapter provides a launch checklist covering the core loop, tools, permissions, context, multi-agent behavior, and persistence.

# 18. Implementation Checklist

Use this as a launch checklist.

## Core

- [ ] Message union exists.
- [ ] Query loop is async generator.
- [ ] Provider messages are normalized at boundaries.
- [ ] Tool-use/tool-result pairing is repaired before API calls.
- [ ] Abort synthesizes missing tool results.
- [ ] Transcript persists user message before model call.

## Tools

- [ ] Every tool has schema.
- [ ] Every tool has permission check.
- [ ] Read-only tools declare concurrency safe.
- [ ] Unsafe tools run serially.
- [ ] Large results persist to disk.
- [ ] Tool progress can stream.

## Permissions

- [ ] Deny rules checked before allow.
- [ ] Tool-specific checks exist.
- [ ] Interactive ask path exists.
- [ ] Headless deny fallback exists.
- [ ] Auto/classifier mode is optional.
- [ ] Audit log records decisions.

## Context

- [ ] Token counting exists.
- [ ] Auto-compact threshold exists.
- [ ] Blocking limit exists.
- [ ] Compact failure circuit breaker exists.
- [ ] Tool result microcompact exists.
- [ ] Max output recovery exists.

## Multi-Agent

- [ ] Subagents have isolated context.
- [ ] Async agents have independent abort controller.
- [ ] Subagents have sidechain transcript.
- [ ] Agent tool filters available tools.
- [ ] Agent tasks are visible/stoppable.
- [ ] Background shell tasks are killed on agent exit.

## Persistence

- [ ] Session transcript.
- [ ] Tool result files.
- [ ] Background task output files.
- [ ] Agent metadata.
- [ ] Resume path.
- [ ] Orphaned permission handling.
