# Agent Framework

A documentation-first blueprint for building reliable single-agent and multi-agent software.

Agent Framework is a practical architecture guide for teams that want to build an agent runtime instead of another thin LLM wrapper. It focuses on the parts that usually make agent systems succeed or fail in production: context management, tool execution, permissions, persistence, replay, multi-agent isolation, cost control, and recovery from broken transcripts.

The repository is organized as a starter knowledge base. One large architecture blueprint has been split into ordered documents so each part can be read, assigned, implemented, and reviewed independently.

## What This Project Gives You

- A concrete architecture for a single-agent coding assistant.
- A path from single-agent reliability to multi-agent orchestration.
- Exact default numbers for context windows, compaction, retries, tool output limits, Bash timeouts, subagent limits, and storage budgets.
- Copy-paste implementation scaffolding: project skeleton, config, interfaces, query loop, and acceptance tests.
- Production guidance for provider adapters, prompt assembly, transcript persistence, SDK events, replay tests, permissions, sandboxing, budgets, and deployment modes.

## Start Here

- [Documentation Index](./docs/README.md)
- [00. Read Me First](./docs/00-read-me-first.md) - optimization history, source map, constants, and drop-in implementation guide.
- [17. Recommended MVP Plans](./docs/17-recommended-mvp-plans.md)
- [30. Deployment Modes And Final Gap Checklist](./docs/30-deployment-modes-and-final-gap-checklist.md)

## Recommended Reading Order

1. [00. Read Me First](./docs/00-read-me-first.md)
2. [02. Core Runtime Model](./docs/02-core-runtime-model.md)
3. [04. Tool Protocol](./docs/04-tool-protocol.md)
4. [07. Context Management](./docs/07-context-management.md)
5. [15. Persistence And Resume](./docs/15-persistence-and-resume.md)
6. [17. Recommended MVP Plans](./docs/17-recommended-mvp-plans.md)
7. [21. Provider Adapter And Model Routing](./docs/21-provider-adapter-and-model-routing.md)
8. [22. Prompt And Context Assembly](./docs/22-prompt-and-context-assembly.md)
9. [26. Evaluation, Replay, And Regression Testing](./docs/26-evaluation-replay-and-regression-testing.md)
10. [29. Multi-Agent Conflict Handling](./docs/29-multi-agent-conflict-handling.md)

## First Milestone

Do not start with autonomous swarms. Start with one reliable local agent loop:

- stream a model response;
- call `Read`, `Grep`, `Glob`, `Bash`, `Edit`, and `Write`;
- enforce permissions before side effects;
- persist transcripts before and after model calls;
- store large tool outputs on disk;
- repair or block invalid `tool_use` / `tool_result` pairs;
- compact context before the provider rejects the request.

Multi-agent behavior should be added only after that foundation is stable.

## Repository Layout

```text
.
├── .agent/
│   ├── config.json
│   ├── permissions.json
│   └── agents/
│       ├── explore.md
│       └── verify.md
├── docs/
│   ├── README.md
│   ├── 00-read-me-first.md
│   ├── 01-product-shape.md
│   └── ...
├── package.json
└── README.md
```

## Useful Commands

```bash
npm run docs:list
npm run docs:check
```

## Who This Is For

This project is useful for:

- engineers designing an agent runtime;
- founders planning an AI coding tool or automation platform;
- teams evaluating whether their agent system has enough safety and persistence;
- researchers comparing single-agent and multi-agent system designs;
- builders who want implementation details instead of broad agent terminology.

## Status

This is currently a documentation-first project. The next natural step is to turn the scaffold in [00. Read Me First](./docs/00-read-me-first.md) into an executable TypeScript runtime.
