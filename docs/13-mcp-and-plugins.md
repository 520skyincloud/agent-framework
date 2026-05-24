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
