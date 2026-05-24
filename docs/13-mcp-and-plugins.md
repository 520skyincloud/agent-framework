> 中文标题：MCP 与插件
>
> English title: MCP And Plugins
>
> 中文导读：本章说明 MCP/插件工具应该作为一等工具接入，包含连接状态、命名冲突、权限、延迟加载和清理规则。
>
> English guide: This chapter explains how MCP/plugin tools should be first-class tools, including connection states, name conflicts, permissions, lazy loading, and cleanup rules.

# 13. MCP And Plugins

MCP/plugin tools should be first-class tools, not special cases in the agent loop.

Recommended MCP connection states:

```ts
type MCPConnection =
  | { type: "pending"; name: string }
  | { type: "connected"; name: string; tools: Tool[]; cleanup(): Promise<void> }
  | { type: "failed"; name: string; error: string }
```

Rules:

- built-in tools win on name conflict,
- sort built-in tools and MCP tools for prompt-cache stability,
- support server-level permissions,
- defer loading large MCP tool schemas,
- allow agent-specific MCP servers,
- clean up inline agent MCP servers when agent exits.
