> 中文标题：动态工具加载
>
> English title: Dynamic Tool Loading
>
> 中文导读：本章说明工具数量过大时如何延迟发送 schema，如何用 ToolSearch 发现工具，以及 alwaysLoad/searchHint/defer_loading 等字段。
>
> English guide: This chapter explains how to defer tool schemas when tool count grows, how ToolSearch discovers tools, and fields like alwaysLoad, searchHint, and defer_loading.

# 14. Dynamic Tool Loading

When tool count grows large, do not send every schema every turn.

Recommended behavior:

1. always send core tools,
2. mark less-used tools as deferred,
3. expose `ToolSearch`,
4. model calls `ToolSearch(query)`,
5. runtime includes discovered schemas in later model calls.

Useful fields:

| Field | Meaning |
|---|---|
| `shouldDefer` | Tool schema can be lazy-loaded |
| `alwaysLoad` | Tool must always be visible |
| `searchHint` | Short keyword description |
| `defer_loading` | Provider-side schema deferral flag |
