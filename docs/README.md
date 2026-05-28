# Agent Framework 文档索引

请按顺序阅读这些文档。它们从一份完整的智能体架构蓝图拆分而来，每个文件都可以单独阅读、分工实现和审查。

## 详细设计

如果你要真正实现一个框架，不要只读 00-30 的总览章节，先进入 [详细设计索引](./detailed-design/00-index.md)。详细设计目录按专题写清楚了状态机、阈值、数据结构、伪代码、失败分支、提示词模板、测试用例和验收标准。

重点入口：

- [运行时主循环](./detailed-design/01-runtime-loop.md)
- [上下文预算与压缩](./detailed-design/03-context-budget-and-compaction.md)
- [模型路由与降级](./detailed-design/04-model-routing-and-downgrade.md)
- [工具执行编排](./detailed-design/07-tool-execution-orchestration.md)
- [多 Agent 编排](./detailed-design/12-multi-agent-orchestration.md)

## 总览章节

- [0. 先读这里](./00-read-me-first.md)
- [1. 产品形态](./01-product-shape.md)
- [2. 核心运行时模型](./02-core-runtime-model.md)
- [3. 会话引擎](./03-session-engine.md)
- [4. 工具协议](./04-tool-protocol.md)
- [5. 工具执行编排](./05-tool-execution-orchestration.md)
- [6. 权限与安全](./06-permissions-and-safety.md)
- [7. 上下文管理](./07-context-management.md)
- [8. 文件处理](./08-file-handling.md)
- [9. 媒体与 Web 限制](./09-media-and-web-limits.md)
- [10. 记忆系统](./10-memory-system.md)
- [11. 多 Agent 架构](./11-multi-agent-architecture.md)
- [12. 任务运行时](./12-task-runtime.md)
- [13. MCP 与插件](./13-mcp-and-plugins.md)
- [14. 动态工具加载](./14-dynamic-tool-loading.md)
- [15. 持久化与恢复](./15-persistence-and-resume.md)
- [16. 可观测性](./16-observability.md)
- [17. 推荐 MVP 路线](./17-recommended-mvp-plans.md)
- [18. 实施检查清单](./18-implementation-checklist.md)
- [19. 建议数据结构](./19-suggested-data-structures.md)
- [20. 值得保留的设计规则](./20-design-rules-worth-keeping.md)
- [21. 模型适配层与模型路由](./21-provider-adapter-and-model-routing.md)
- [22. Prompt 与上下文组装](./22-prompt-and-context-assembly.md)
- [23. 配置系统与目录结构](./23-configuration-and-directory-layout.md)
- [24. 存储 Schema 与恢复算法](./24-storage-schema-and-resume-algorithm.md)
- [25. SDK 与 UI 事件协议](./25-sdk-and-ui-event-protocol.md)
- [26. 评估、回放与回归测试](./26-evaluation-replay-and-regression-testing.md)
- [27. 安全边界](./27-security-boundary.md)
- [28. 预算、速率限制与配额控制](./28-budget-rate-limit-and-quota-control.md)
- [29. 多 Agent 冲突处理](./29-multi-agent-conflict-handling.md)
- [30. 部署模式与最终检查清单](./30-deployment-modes-and-final-gap-checklist.md)
