# 30. Deployment Modes And Final Gap Checklist

## 30.1 Deployment Modes

| Mode | Architecture |
|---|---|
| Local CLI | single process, local files, local shell |
| Desktop app | local process + GUI event stream |
| SDK/headless | no interactive prompts; ask becomes deny unless caller handles control events |
| Remote session | server worker + WebSocket SDK events + HTTP control messages |
| Team platform | multi-user auth, central policy, remote workers, per-user quotas |

Mode-specific defaults:

| Setting | CLI | SDK | Remote |
|---|---:|---:|---:|
| Interactive permission ask | yes | caller-provided | WebSocket control |
| Non-stream fallback timeout | `300_000 ms` | `300_000 ms` | `120_000 ms` |
| Background tasks | local process | opt-in | worker-managed |
| Transcript location | local disk | caller-configured | server storage |
| Default ask fallback | ask user | deny | send control request |

## 30.2 Final Gap Checklist

After the second 10-round pass, a serious Agent framework spec should include all of these:

- [ ] Provider adapter with normalized streaming events.
- [ ] Retry policy with concrete backoff, 429/529 behavior, stream idle timeout.
- [ ] Deterministic prompt assembly order.
- [ ] Context budget split and omission records.
- [ ] Typed config schema and precedence.
- [ ] Runtime directory layout.
- [ ] Transcript JSONL schema.
- [ ] Resume repair algorithm.
- [ ] SDK event and control protocol.
- [ ] Golden replay test harness.
- [ ] Permission audit log.
- [ ] Sandbox and secret handling rules.
- [ ] Budget limits for tokens, tool calls, agents, tasks, storage.
- [ ] File lease and worktree merge protocol.
- [ ] Deployment-mode-specific behavior.

Final rule:

```text
If an Agent framework cannot replay a past run, repair a broken transcript,
explain a permission decision, and bound its own cost, it is still a demo,
not a framework.
```
