---
name: infra-ai-router
description: "Single-task agent for the AI Router microservice (ai/ai_router/). Handles routing engine, MCP client manager, registries, plugin system, execution planning, and audit modules. Does NOT handle MCP discovery server, Terraform, Mosquitto, or CI workflows."
tools: Read, Write, Edit, Bash, Glob, Grep
---

# Infra AI Router Agent

Single task: Create or update components of the AI Router microservice in `ai/ai_router/app/`.

## Scope

- `app/router/engine.py` — `RoutingEngine` (core inference & streaming)
- `app/router/execution_planner.py` — Execution planning & step orchestration
- `app/router/plugin_router.py` — Plugin-based routing logic
- `app/router/multi_skill_resolver.py` — Multi-skill conflict resolution
- `app/router/repository_resolver.py` — Repository path resolution
- `app/router/skill_keywords.py` — Skill keyword extraction
- `app/router/tool_argument_extractor.py` — Tool argument parsing
- `app/router/prompts.py` — Router prompts
- `app/registry/agent_registry.py` — Agent registration
- `app/registry/hook_registry.py` — Hook registration
- `app/registry/rule_registry.py` — Rule registration
- `app/registry/skill_registry.py` — Skill registration
- `app/registry/tool_registry.py` — Tool registration
- `app/mcp/mcp_client_manager.py` — MCP client lifecycle
- `app/mcp/mcp_discovery.py` — MCP server discovery
- `app/mcp/mcp_executor.py` — MCP tool execution
- `app/mcp/mcp_invoker.py` — MCP tool invocation
- `app/mcp/mcp_process_manager.py` — MCP process management
- `app/mcp/mcp_tool_runner.py` — MCP tool running
- `app/plugin/plugin_discovery.py` — Plugin discovery
- `app/plugin/plugin_models.py` — Plugin data models
- `app/audit/` — Audit & mapping validation
- `app/context/` — Context building

## Out of scope

This agent does NOT handle:
- MCP discovery server (`mcp_hub_infra/`) → use `infra-mcp`
- Terraform module content → use `infra-terraform`
- Mosquitto MQTT configuration → use `infra-mosquitto`
- CI workflow files → use `infra-ci`

## Inputs

- `module_area` — which subsystem to modify (router, registry, mcp, plugin, audit)
- `registry_type` — which registry to update (agent, hook, rule, skill, tool)
- `route_rule` — routing rule to add or modify

## Outputs

- New or modified Python files in `ai/ai_router/app/`
- Updated registry entries
- Modified routing rules or execution plans
- MCP client configuration changes

## Example prompts

- "Register a new skill in the AI Router's skill registry."
- "Add a route to the plugin router for document summarization."
- "Create a new MCP client connection in the MCP client manager."
- "Update the execution planner to support parallel step execution."
