# Agent Framework（智能体框架）

一个用于构建 **单智能体软件** 和 **多智能体系统** 的架构文档与实施蓝图。

Agent Framework 不是一个泛泛讲概念的 Agent 项目，而是一份偏工程落地的框架说明。它关注真正做智能体系统时最容易出问题的部分：上下文管理、工具调用、权限控制、会话持久化、失败恢复、多 Agent 隔离、成本预算、回放测试和生产部署。

这个仓库目前是 **文档优先** 的项目：它把一份完整的大型架构文档拆成了有顺序的多个章节，方便阅读、分工、实现和审查。后续可以基于这些文档继续实现成一个 TypeScript 智能体运行时。

## 这个项目提供什么

- 一套具体的单 Agent 编程助手架构。
- 一条从单 Agent 稳定性走向多 Agent 编排的路线。
- 明确的默认数字：上下文窗口、压缩阈值、重试次数、工具输出限制、Bash 超时、子 Agent 限制、存储预算等。
- 可以直接复制使用的工程脚手架：项目目录、配置文件、接口定义、主循环示例、验收测试。
- 面向生产环境的设计：模型适配层、Prompt 组装、Transcript 持久化、SDK 事件流、回放测试、权限审计、安全沙箱、预算控制和部署模式。

## 从这里开始

- [文档索引](./docs/README.md)
- [00. 先读这里](./docs/00-read-me-first.md)：优化记录、源码映射、关键常量、开箱即用实施指南。
- [详细设计索引](./docs/detailed-design/00-index.md)：如果你要实现框架，先读这里。这里按专题拆好了主循环、消息、上下文、模型路由、工具、权限、持久化、多 Agent、评估和部署。
- [17. 推荐 MVP 路线](./docs/17-recommended-mvp-plans.md)
- [30. 部署模式与最终检查清单](./docs/30-deployment-modes-and-final-gap-checklist.md)

## 推荐阅读顺序

1. [00. 先读这里](./docs/00-read-me-first.md)
2. [详细设计索引](./docs/detailed-design/00-index.md)
3. [02. 核心运行时模型](./docs/02-core-runtime-model.md)
4. [04. 工具协议](./docs/04-tool-protocol.md)
5. [07. 上下文管理](./docs/07-context-management.md)
6. [15. 持久化与恢复](./docs/15-persistence-and-resume.md)
7. [17. 推荐 MVP 路线](./docs/17-recommended-mvp-plans.md)
8. [21. 模型适配层与模型路由](./docs/21-provider-adapter-and-model-routing.md)
9. [22. Prompt 与上下文组装](./docs/22-prompt-and-context-assembly.md)
10. [26. 评估、回放与回归测试](./docs/26-evaluation-replay-and-regression-testing.md)
11. [29. 多 Agent 冲突处理](./docs/29-multi-agent-conflict-handling.md)

## 第一阶段目标

不要一上来就做自主 swarm 或复杂多 Agent。第一阶段只做一个可靠的本地单 Agent 主循环：

- 能流式返回模型输出；
- 能调用 `Read`、`Grep`、`Glob`、`Bash`、`Edit`、`Write`；
- 所有副作用执行前必须经过权限系统；
- 模型调用前后都要持久化 transcript；
- 大型工具输出要落盘，不能直接塞进上下文；
- 永远不能发送不合法的 `tool_use` / `tool_result` 配对；
- 在上下文接近限制前主动 compact。

多 Agent 能力应该建立在这个稳定基础之上。

## 仓库结构

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
│   ├── detailed-design/
│   └── ...
├── package.json
└── README.md
```

## 可用命令

```bash
npm run docs:list
npm run docs:check
```

## 适合谁

这个项目适合：

- 想设计智能体运行时的工程师；
- 想做 AI 编程工具或自动化平台的创业者；
- 想评估自己 Agent 系统是否足够可靠的团队；
- 想研究单 Agent 和多 Agent 系统差异的人；
- 不想只看宽泛概念、而是需要具体实现细节的开发者。

## 当前状态

当前仓库是文档优先项目，还不是可执行框架。下一步最自然的方向，是把 [00. 先读这里](./docs/00-read-me-first.md) 里的工程脚手架实现成一个可运行的 TypeScript Agent Runtime。
