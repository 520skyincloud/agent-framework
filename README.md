# Agent Framework

一个用于构建 **单智能体软件** 和 **多智能体系统** 的架构文档与实施蓝图。

A documentation-first architecture and implementation blueprint for building **single-agent software** and **multi-agent systems**.

Agent Framework 不是一个泛泛讲概念的 Agent 项目，而是一份偏工程落地的框架说明。它关注真正做智能体系统时最容易出问题的部分：上下文管理、工具调用、权限控制、会话持久化、失败恢复、多 Agent 隔离、成本预算、回放测试和生产部署。

Agent Framework is not a vague conceptual agent project. It is an engineering-oriented framework guide focused on the parts that usually make agent systems succeed or fail: context management, tool execution, permissions, persistence, failure recovery, multi-agent isolation, cost control, replay testing, and production deployment.

这个仓库目前是 **文档优先** 的项目：它把一份完整的大型架构文档拆成了有顺序的多个章节，方便阅读、分工、实现和审查。后续可以基于这些文档继续实现成一个 TypeScript 智能体运行时。

This repository is currently **documentation-first**. A large architecture blueprint has been split into ordered chapters so each part can be read, assigned, implemented, and reviewed independently. The next step is to turn the blueprint into an executable TypeScript agent runtime.

## 这个项目提供什么

## What This Project Gives You

- 一套具体的单 Agent 编程助手架构。
- 一条从单 Agent 稳定性走向多 Agent 编排的路线。
- 明确的默认数字：上下文窗口、压缩阈值、重试次数、工具输出限制、Bash 超时、子 Agent 限制、存储预算等。
- 可以直接复制使用的工程脚手架：项目目录、配置文件、接口定义、主循环示例、验收测试。
- 面向生产环境的设计：模型适配层、Prompt 组装、Transcript 持久化、SDK 事件流、回放测试、权限审计、安全沙箱、预算控制和部署模式。

- A concrete architecture for a single-agent coding assistant.
- A path from single-agent reliability to multi-agent orchestration.
- Exact default numbers for context windows, compaction thresholds, retry counts, tool output limits, Bash timeouts, subagent limits, and storage budgets.
- Copy-paste implementation scaffolding: project layout, config, interfaces, query loop, and acceptance tests.
- Production-oriented guidance for provider adapters, prompt assembly, transcript persistence, SDK events, replay testing, permission audits, sandboxing, budget control, and deployment modes.

## 从这里开始

## Start Here

- [文档索引](./docs/README.md)
- [00. 先读这里](./docs/00-read-me-first.md)：优化记录、源码映射、关键常量、开箱即用实施指南。
- [17. 推荐 MVP 路线](./docs/17-recommended-mvp-plans.md)
- [30. 部署模式与最终检查清单](./docs/30-deployment-modes-and-final-gap-checklist.md)

- [Documentation Index](./docs/README.md)
- [00. Read Me First](./docs/00-read-me-first.md): optimization history, source map, constants, and drop-in implementation guide.
- [17. Recommended MVP Plans](./docs/17-recommended-mvp-plans.md)
- [30. Deployment Modes And Final Gap Checklist](./docs/30-deployment-modes-and-final-gap-checklist.md)

## 推荐阅读顺序

## Recommended Reading Order

1. [00. 先读这里](./docs/00-read-me-first.md)
2. [02. 核心运行时模型](./docs/02-core-runtime-model.md)
3. [04. 工具协议](./docs/04-tool-protocol.md)
4. [07. 上下文管理](./docs/07-context-management.md)
5. [15. 持久化与恢复](./docs/15-persistence-and-resume.md)
6. [17. 推荐 MVP 路线](./docs/17-recommended-mvp-plans.md)
7. [21. 模型适配层与模型路由](./docs/21-provider-adapter-and-model-routing.md)
8. [22. Prompt 与上下文组装](./docs/22-prompt-and-context-assembly.md)
9. [26. 评估、回放与回归测试](./docs/26-evaluation-replay-and-regression-testing.md)
10. [29. 多 Agent 冲突处理](./docs/29-multi-agent-conflict-handling.md)

## 第一阶段目标

## First Milestone

不要一上来就做自主 swarm 或复杂多 Agent。第一阶段只做一个可靠的本地单 Agent 主循环：

Do not start with autonomous swarms or complex multi-agent behavior. The first milestone is one reliable local single-agent loop:

- 能流式返回模型输出；
- 能调用 `Read`、`Grep`、`Glob`、`Bash`、`Edit`、`Write`；
- 所有副作用执行前必须经过权限系统；
- 模型调用前后都要持久化 transcript；
- 大型工具输出要落盘，不能直接塞进上下文；
- 永远不能发送不合法的 `tool_use` / `tool_result` 配对；
- 在上下文接近限制前主动 compact。

- stream model output;
- call `Read`, `Grep`, `Glob`, `Bash`, `Edit`, and `Write`;
- enforce permissions before side effects;
- persist transcripts before and after model calls;
- store large tool outputs on disk instead of putting them directly into context;
- never send invalid `tool_use` / `tool_result` pairs;
- compact context before the provider rejects the request.

多 Agent 能力应该建立在这个稳定基础之上。

Multi-agent capability should be built on top of this stable foundation.

## 仓库结构

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

## 可用命令

## Useful Commands

```bash
npm run docs:list
npm run docs:check
```

## 适合谁

## Who This Is For

这个项目适合：

- 想设计智能体运行时的工程师；
- 想做 AI 编程工具或自动化平台的创业者；
- 想评估自己 Agent 系统是否足够可靠的团队；
- 想研究单 Agent 和多 Agent 系统差异的人；
- 不想只看宽泛概念、而是需要具体实现细节的开发者。

This project is useful for:

- engineers designing an agent runtime;
- founders building AI coding tools or automation platforms;
- teams evaluating whether their agent systems are reliable enough;
- researchers comparing single-agent and multi-agent system designs;
- developers who need concrete implementation details instead of broad terminology.

## 当前状态

## Status

当前仓库是文档优先项目，还不是可执行框架。下一步最自然的方向，是把 [00. 先读这里](./docs/00-read-me-first.md) 里的工程脚手架实现成一个可运行的 TypeScript Agent Runtime。

This is currently a documentation-first project, not yet an executable framework. The next natural step is to implement the scaffold in [00. Read Me First](./docs/00-read-me-first.md) as a runnable TypeScript Agent Runtime.
