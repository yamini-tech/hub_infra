---
name: infra-mcp
description: Create or update MCP (Model Context Protocol) infrastructure discovery server tools and configuration in the mcp_hub_infra/ TypeScript project. Use when adding new discovery tools, updating existing tool logic, or modifying MCP server registration (.vscode/mcp.json).
metadata:
  model: models/gemini-3.1-pro-preview
  last_modified: Thu, 02 Jul 2026 00:00:00 GMT
---

# Infra MCP Skill

## Contents
- [Core Guidelines](#core-guidelines)
- [Workflow: Adding a New MCP Tool](#workflow-adding-a-new-mcp-tool)
- [Workflow: Modifying Server Configuration](#workflow-modifying-server-configuration)
- [Examples](#examples)

## Core Guidelines

- **Tool Isolation**: Each tool lives in its own file under `mcp_hub_infra/src/tools/<name>.ts`. The file exports a single async handler function returning `{ content: [...] }`.
- **Tool Registration**: Every new tool must be registered in `mcp_hub_infra/src/server.ts` with a unique snake_case name, a clear description (for LLM dispatch), and an optional zod input schema.
- **Error Handling**: Tool handlers must catch errors and return structured error responses with `isError: true`. Never let uncaught exceptions reach the MCP client.
- **TypeScript Standards**: Use strict TypeScript with ESM modules. Import `HUB_INFRA_PATH` from `../config.js` for workspace root access.
- **Config Parity**: When adding tools that connect to external services, update `.vscode/mcp.json` with the required environment variables.

## Workflow: Adding a New MCP Tool

Use this checklist to implement a new discovery or utility tool for the MCP server.

**Task Progress:**
- [ ] 1. Define the tool's purpose, whether it needs input parameters, and its return data shape.
- [ ] 2. Create `mcp_hub_infra/src/tools/<name>.ts` with the exported async handler function.
- [ ] 3. Implement the handler using `fs` for file system access and `HUB_INFRA_PATH` for the workspace root.
- [ ] 4. Register the tool in `mcp_hub_infra/src/server.ts` with name, description, and zod schema if needed.
- [ ] 5. If the tool requires new environment variables, update `.vscode/mcp.json`.
- [ ] 6. Build and verify: `npm run build` passes with no TypeScript errors.
- [ ] 7. **Feedback Loop**: Check MCP server output for tool discovery -> review errors -> fix registration or type mismatches -> rebuild.

1. **Define the Tool**: Determine what infrastructure data the tool returns (Terraform modules, CI/CD workflows, env vars, etc.) and whether it needs a search parameter.
2. **Implement the Handler**: Create the TypeScript file. Use `fs.readdirSync` / `fs.statSync` for file system exploration (see `discoverInfraArchitecture.ts` for static data or `findResource.ts` for recursive search).
3. **Register in Server**: Add the import and `server.tool()` call in `server.ts`. The description must clearly tell an LLM when to invoke this tool.
4. **Update Config**: If the tool needs workspace path config, ensure `HUB_INFRA_PATH` is set in `config.ts` and `.vscode/mcp.json`.
5. **Test**: Run `npm run build` to verify compilation.

## Workflow: Modifying Server Configuration

Use this checklist when updating MCP server registration details (command, args, environment variables).

**Task Progress:**
- [ ] 1. Identify which config files need updating: `.vscode/mcp.json`.
- [ ] 2. Update the `command` and `args` arrays if the server executable or arguments change.
- [ ] 3. Add, remove, or update environment variables in the `env` block. Use `{env:VAR_NAME}` interpolation for secrets.
- [ ] 4. Verify the JSON is valid (no trailing commas, valid string values).
- [ ] 5. Test that the MCP server starts and connects successfully.

## Examples

### High-Fidelity Implementation: Creating a New Static Discovery Tool

**Tool File (`mcp_hub_infra/src/tools/discoverInfraArchitecture.ts`):**
```typescript
export async function discoverInfraArchitectureHandler() {
  return {
    content: [
      {
        type: "text" as const,
        text: JSON.stringify(
          {
            infrastructure: "Terraform",
            messaging: "Mosquitto MQTT",
            ciCd: "GitHub Actions",
            structure: [
              "terraform/",
              "terraform/modules/",
              "mosquitto/",
              "workflows/",
            ],
          },
          null,
          2
        ),
      },
    ],
  };
}
```

**Registration in `server.ts`:**
```typescript
import { discoverInfraArchitectureHandler } from "./tools/discoverInfraArchitecture.js";

server.tool(
  "discover_infra_architecture",
  "Discover infrastructure architecture and repository structure",
  {},
  discoverInfraArchitectureHandler
);
```

### High-Fidelity Implementation: Creating a Parameterized Search Tool

**Tool File (`mcp_hub_infra/src/tools/findResource.ts`):**
```typescript
import * as fs from "fs";
import path from "path";
import { HUB_INFRA_PATH } from "../config.js";

export async function findResourceHandler({
  resource,
}: {
  resource: string;
}) {
  const root = path.join(HUB_INFRA_PATH, "app");
  const resources: string[] = [];

  const searchDirectory = (dir: string, depth: number = 0) => {
    if (depth > 5) return;
    try {
      const files = fs.readdirSync(dir);
      for (const file of files) {
        const fullPath = path.join(dir, file);
        const stat = fs.statSync(fullPath);
        if (stat.isDirectory() && !file.startsWith(".")) {
          if (file.toLowerCase().includes(resource.toLowerCase())) {
            resources.push(path.relative(HUB_INFRA_PATH, fullPath));
          }
          searchDirectory(fullPath, depth + 1);
        } else if (stat.isFile()) {
          if (file.toLowerCase().includes(resource.toLowerCase())) {
            resources.push(path.relative(HUB_INFRA_PATH, fullPath));
          }
        }
      }
    } catch (error) {}
  };

  searchDirectory(root);

  return {
    content: [
      {
        type: "text" as const,
        text: JSON.stringify(
          { resource, found: resources.length > 0, locations: resources },
          null,
          2
        ),
      },
    ],
  };
}
```

**Registration in `server.ts`:**
```typescript
import { z } from "zod";
import { findResourceHandler } from "./tools/findResource.js";

server.tool(
  "find_resource",
  "Find infrastructure resources and related files",
  { resource: z.string() },
  findResourceHandler
);
```
