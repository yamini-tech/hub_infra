---
name: infra-ai-router
description: Create or update AI Router microservice components in ai/ai_router/ including routing engine, MCP client manager, registries, plugin system, execution planning, and audit modules. Use when adding routing rules, registering plugins, managing MCP clients, or modifying the AI inference pipeline.
metadata:
  model: models/gemini-3.1-pro-preview
  last_modified: Thu, 02 Jul 2026 00:00:00 GMT
---

# Infra AI Router Skill

## Contents
- [Core Guidelines](#core-guidelines)
- [Workflow: Registering a New Skill in the Registry](#workflow-registering-a-new-skill-in-the-registry)
- [Workflow: Adding a Plugin Route to the AI Router](#workflow-adding-a-plugin-route-to-the-ai-router)
- [Examples](#examples)

## Core Guidelines

- **Engine as Entry Point**: The `RoutingEngine` (`app/router/engine.py`) is the core inference orchestrator. All routing logic flows through `engine.process_stream(messages)`. Never bypass the engine to call downstream components directly.
- **Registry Isolation**: Each registry (`agent`, `hook`, `rule`, `skill`, `tool`) lives in its own file under `app/registry/`. Registries follow a uniform interface: `build()` (loads from plugin discovery), `get(repository, id)`, `get_all()`, `get_repository_*(repository)`, and `repositories()`. Keep registries focused on a single entity type.
- **MCP Client Lifecycle**: The `MCPClientManager` in `app/mcp/mcp_client_manager.py` handles server connection lifecycles. Use `get_session(server_name, command, working_dir)` to obtain a `ClientSession`, and `shutdown()` to close all sessions via `stack.aclose()`. Never manage MCP processes manually.
- **Plugin Discovery**: Plugins are auto-discovered via `app/plugin/plugin_discovery.py`. New plugins must define their capabilities in `plugin_models.py` and be placed in the expected discovery path.
- **Async Everything**: All router, MCP, and inference operations use async/await. Use `async for` to consume streaming responses. Avoid `time.sleep()` — use `asyncio.sleep()`.

## Workflow: Registering a New Skill in the Registry

Use this checklist to add a new skill entry to the AI Router's skill registry.

**Task Progress:**
- [ ] 1. Determine the skill's name, version, description, and capabilities.
- [ ] 2. Read `app/registry/skill_registry.py` to understand the existing entry format.
- [ ] 3. Add the new skill entry following the same structure (name, metadata, handler reference).
- [ ] 4. If the skill needs routing rules, update `app/router/plugin_router.py` or `app/router/multi_skill_resolver.py`.
- [ ] 5. Test by initializing the `RoutingEngine` and verifying the skill is discoverable via `get_all()`.
- [ ] 6. **Feedback Loop**: Query the registry -> verify new skill appears -> test routing to the new skill -> adjust if needed.

1. **Define Skill**: Choose a unique name, set a semver version, and write a clear description that tells the router when to invoke this skill.
2. **Read Registry Pattern**: Open `app/registry/skill_registry.py` and note the entry structure (dict keys, metadata fields).
3. **Add Entry**: Append the new skill to the registry's data store. Include all required metadata fields.
4. **Wire Routing**: If the skill is triggered by specific keywords or patterns, add corresponding rules to the plugin router or multi-skill resolver.
5. **Verify**: Import the engine, instantiate it, and call the registry's `list_all()` to confirm the new skill is registered.

## Workflow: Adding a Plugin Route to the AI Router

Use this checklist to add a new routing path through the AI Router's plugin system.

**Task Progress:**
- [ ] 1. Define the route trigger (user intent, keywords, or message pattern).
- [ ] 2. Create or update the plugin model in `app/plugin/plugin_models.py`.
- [ ] 3. Add the route to `app/router/plugin_router.py` with the trigger condition and target plugin.
- [ ] 4. Ensure the target plugin is discoverable via `app/plugin/plugin_discovery.py`.
- [ ] 5. Test by sending a message that matches the trigger and verifying the route is taken.
- [ ] 6. **Feedback Loop**: Submit test prompts -> trace routing decisions -> verify plugin is invoked -> adjust trigger conditions.

## Examples

### High-Fidelity Implementation: Running the Routing Engine

**Entry Point (`ai/ai_router/app/main.py`):**
```python
from app.router.engine import RoutingEngine
from app.mcp.mcp_client_manager import mcp_client_manager

engine = RoutingEngine()
messages = [{"role": "user", "content": "Summarize this document"}]

async for token in engine.process_stream(messages):
    print(token, end="", flush=True)

await mcp_client_manager.shutdown()
```

### High-Fidelity Implementation: Registry Entry Pattern

**Skill Registry (`app/registry/skill_registry.py`):**
```python
class SkillRegistry:
    def __init__(self) -> None:
        self._skills: Dict[str, SkillDefinition] = {}
        self._repo_skill_index: Dict[str, Dict[str, SkillDefinition]] = {}

    def build(self) -> None:
        discovery = discover_all()
        self._skills.clear()
        self._repo_skill_index.clear()
        for repo in discovery.repositories:
            repo_name = repo.repository
            if repo_name not in self._repo_skill_index:
                self._repo_skill_index[repo_name] = {}
            for skill in repo.skills:
                manifest_path = skill.path / ".plugin" / "plugin.json"
                if not manifest_path.exists():
                    continue
                with open(manifest_path, "r", encoding="utf-8") as file:
                    raw = json.load(file)
                definition = SkillDefinition(
                    repository=repo_name,
                    skill_id=raw["id"],
                    name=raw.get("name", raw["id"]),
                    mcp_server=raw.get("mcp_server", ""),
                    tool=raw.get("tool", ""),
                    enabled=raw.get("enabled", True),
                    path=skill.path,
                )
                unique_key = f"{repo_name}:{definition.skill_id}"
                self._skills[unique_key] = definition
                self._repo_skill_index[repo_name][definition.skill_id] = definition

    def get(self, repository: str, skill_id: str) -> Optional[SkillDefinition]:
        return self._repo_skill_index.get(repository, {}).get(skill_id)

    def get_all(self) -> List[SkillDefinition]:
        return list(self._skills.values())

    def get_repository_skills(self, repository: str) -> List[SkillDefinition]:
        return list(self._repo_skill_index.get(repository, {}).values())

    def repositories(self) -> List[str]:
        return list(self._repo_skill_index.keys())

skill_registry = SkillRegistry()
```

### High-Fidelity Implementation: MCP Client Manager Session

**MCP Client Manager (`app/mcp/mcp_client_manager.py`):**
```python
class MCPClientManager:
    def __init__(self):
        self._sessions = {}

    async def get_session(self, server_name: str, command: list[str], working_dir: str):
        managed = self._sessions.get(server_name)
        if managed:
            try:
                await managed.session.list_tools()
                return managed.session
            except Exception:
                try:
                    await managed.stack.aclose()
                except Exception:
                    pass
                self._sessions.pop(server_name, None)
        return await self._create_session(server_name, command, working_dir)

    async def shutdown(self):
        sessions = list(self._sessions.values())
        self._sessions.clear()
        for managed in sessions:
            try:
                await managed.stack.aclose()
            except Exception:
                pass

mcp_client_manager = MCPClientManager()
```
