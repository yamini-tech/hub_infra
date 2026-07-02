---
mode: agent
agent: infra-mcp
name: infra-mcp-prompt
description: "Prompt for the infra-mcp agent. Creates and updates MCP discovery tools in mcp_hub_infra/src/tools/, handles server registration in server.ts, and manages VS Code environment config."
---

### Requirements

1. **Tool Isolation:** Each tool lives in its own file under `mcp_hub_infra/src/tools/<name>.ts`. The file exports a single async handler function.
2. **Response Format:** Handlers return `{ content: [{ type: "text" as const, text: JSON.stringify(data, null, 2) }] }`. On error return `{ content: [{ type: "text", text: "Error: ..." }], isError: true }`.
3. **Tool Registration:** Register every tool in `mcp_hub_infra/src/server.ts` with a unique snake_case name, a description that tells the LLM when to invoke it, and an optional zod input schema.
4. **Parameterized Tools:** Use `z.object({ param: z.string() })` for tools that accept input (see `findResource.ts` pattern).
5. **Config Import:** Tools access the workspace root via `import { HUB_INFRA_PATH } from "../config.js"`.
6. **Build Verification:** Run `npm run build` after changes to confirm compilation.

### Constraints

- ESM modules with strict TypeScript — no `require()`, no `any` types
- Snake_case for tool names in `server.tool()` (e.g., `"discover_terraform_modules"`)
- Follow existing handler signatures in the 9 upstream tools
- Environment vars added to `.vscode/mcp.json` under the server's `env` block

### Success Criteria

- `npm run build` passes with no TypeScript errors
- New tool appears in server capabilities when MCP server starts
- Parameterized tools accept and use input correctly
- Existing tools are not broken by changes

### Usage Template

```
Add/modify an MCP discovery tool:
- Tool file: mcp_hub_infra/src/tools/<name>.ts
- Parameters: [list params with types or "none"]
- Description: [LLM-facing description]
- Registration in server.ts with name [snake_case_name]
Show the diff and wait for my confirmation before applying.
```

### Chat Example

```
User: Add a discover_networking tool that returns VPC, subnet, and security group info.
```

Agent (expected):
- Creates `mcp_hub_infra/src/tools/discoverNetworking.ts` with the handler
- Registers `server.tool("discover_networking", "Discover networking resources", {}, handler)`
- Runs `npm run build` to verify
- Shows diff and waits for confirmation
