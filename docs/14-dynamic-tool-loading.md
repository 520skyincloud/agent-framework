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
