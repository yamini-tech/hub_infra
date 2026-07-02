---
mode: agent
agent: infra-ai-router
name: infra-ai-router-prompt
description: "Prompt for the infra-ai-router agent. Creates and updates AI Router microservice components including routing engine, MCP client manager, registries, plugin system, execution planning, and audit modules."
---

### Requirements

1. **Routing Engine:** The `RoutingEngine` in `app/router/engine.py` is the core entry point. It exposes `process_stream(messages)` which yields response tokens asynchronously.
2. **MCP Client Lifecycle:** The `mcp_client_manager` in `app/mcp/mcp_client_manager.py` manages MCP server connections. Always call `await mcp_client_manager.shutdown()` during cleanup.
3. **Registry Pattern:** Registries in `app/registry/` expose `build()` (loads from plugin discovery), `get(repository, id)`, `get_all()`, `get_repository_*(repository)`, and `repositories()`. No registries expose `register()`/`unregister()` — entries are built from plugin manifests.
4. **Plugin System:** Plugins are discovered via `plugin_discovery.py` and modeled with `plugin_models.py`. New plugins must implement the expected plugin interface.
5. **Async Pattern:** All I/O operations (MCP calls, file reads, inference) must use async/await. Use `async for` for streaming responses from the engine.
6. **Audit Module:** Changes to routing or plugin logic should include corresponding audit entries in `app/audit/` for traceability.

### Constraints

- Python 3.10+ with async/await — no synchronous blocking calls in hot paths
- Follow existing module structure and import patterns in `ai/ai_router/app/`
- Registry entries must include unique names and descriptive metadata
- MCP client config must include server name, command, and args

### Success Criteria

- Module imports without errors
- Registry operations (build/get/get_all/repositories) work correctly
- MCP client connects and disconnects cleanly
- Routing engine processes messages end-to-end
- Tests pass if present

### Usage Template

```
Create/update an AI Router component:
- Module area: [router/registry/mcp/plugin/audit]
- Component: [specific file or class]
- Change: [description of what to add/modify]
Show the diff and wait for my confirmation before applying.
```

### Chat Example

```
User: Register a new "document-summarization" skill in the AI Router's skill registry.
```

Agent (expected):
- Reads existing `app/registry/skill_registry.py` to understand the pattern
- Adds the new skill entry with name, version, and description
- Shows diff and waits for confirmation
