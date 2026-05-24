# 19. Suggested Data Structures

## QueryParams

```ts
type QueryParams = {
  messages: Message[]
  systemPrompt: SystemPrompt
  userContext: Record<string, string>
  systemContext: Record<string, string>
  tools: Tool[]
  canUseTool: CanUseToolFn
  toolUseContext: ToolUseContext
  model: string
  fallbackModel?: string
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number; remaining?: number }
  querySource: QuerySource
}
```

## ToolUseContext

```ts
type ToolUseContext = {
  options: {
    tools: Tool[]
    commands: Command[]
    mainLoopModel: string
    thinkingConfig: ThinkingConfig
    mcpClients: MCPConnection[]
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitions
    querySource?: QuerySource
  }

  abortController: AbortController
  messages: Message[]
  readFileState: FileStateCache

  getAppState(): AppState
  setAppState(fn: (prev: AppState) => AppState): void
  setAppStateForTasks?: (fn: (prev: AppState) => AppState) => void

  agentId?: string
  agentType?: string
  toolUseId?: string
  queryTracking?: { chainId: string; depth: number }
}
```

## AppState

```ts
type AppState = {
  toolPermissionContext: ToolPermissionContext
  mcp: {
    clients: MCPConnection[]
    tools: Tool[]
  }
  tasks: Record<string, TaskState>
  todos: Record<string, Todo[]>
  agentDefinitions: AgentDefinitions
  agentNameRegistry: Map<string, string>
  effortValue?: string
  fastMode?: boolean
}
```
