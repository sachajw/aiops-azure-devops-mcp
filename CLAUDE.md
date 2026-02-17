# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the Azure DevOps MCP (Model Context Protocol) Server - a TypeScript project that exposes Azure DevOps functionality as tools for LLM-based agents. It provides a thin abstraction layer over the Azure DevOps REST APIs.

## Build and Development Commands

```bash
# Install dependencies
npm install

# Build the project
npm run build

# Watch mode for development
npm run watch

# Run all tests
npm test

# Run a specific test file
npm test test/src/tools/core.test.ts

# Run tests with coverage
npm test -- --coverage

# Lint and format
npm run eslint
npm run eslint-fix
npm run format
npm run format-check

# Validate tool names and parameters (run after adding new tools)
npm run validate-tools

# Clean build artifacts
npm run clean

# Run MCP inspector for local testing
npm run inspect
```

## Architecture

### Entry Point and Server Setup

- **`src/index.ts`**: Main entry point. Parses CLI arguments (organization name, domains, authentication type), creates the MCP server, and connects via stdio transport.
- **`src/tools.ts`**: Orchestrates tool registration by domain. Each domain has its own `configure*Tools` function.

### Domain-Based Tool Organization

Tools are organized into domains defined in `src/shared/domains.ts`:

- `core` - Projects, teams, identities
- `work` - Iterations, capacity
- `work-items` - Work item CRUD, queries, comments
- `repositories` - Repos, branches, pull requests, commits
- `pipelines` - Builds, runs, definitions
- `wiki` - Wiki pages
- `test-plans` - Test plans, suites, cases
- `search` - Code, wiki, work item search
- `advanced-security` - Security alerts

Domains can be enabled via CLI: `node dist/index.js <org> -d core work work-items`

### Tool Implementation Pattern

Each tool module (e.g., `src/tools/core.ts`) follows this pattern:

1. Define tool name constants:

```typescript
const CORE_TOOLS = {
  list_projects: "core_list_projects",
};
```

2. Register tools using `server.tool()` with Zod schemas:

```typescript
server.tool(TOOL_NAME, "Description", { param: z.string().describe("...") }, async ({ param }) => {
  const connection = await connectionProvider();
  const api = await connection.getSomeApi();
  // ... implementation
  return { content: [{ type: "text", text: JSON.stringify(result) }] };
});
```

### Authentication

Authentication is handled in `src/auth.ts` with three modes:

- `interactive` (default) - OAuth via browser popup
- `azcli` - Azure CLI credentials
- `envvar` - Token from `ADO_MCP_AUTH_TOKEN` environment variable

### Logging

All logs must go to **stderr** (never stdout, which would break MCP protocol). Use the winston logger from `src/logger.ts`:

```typescript
import { logger } from "./logger.js";
logger.info("message", { metadata });
```

Set `LOG_LEVEL` environment variable to control verbosity (`error`, `warn`, `info`, `debug`).

## Tool Naming Conventions

Tool names must match pattern `^[a-zA-Z0-9_.-]{1,64}$` and follow the format: `<domain>_<action>`

Examples: `core_list_projects`, `wit_get_work_item`, `repo_create_branch`

## Testing

Tests use Jest with `ts-jest` and are located in `test/` mirroring `src/` structure. Tests mock:

- `McpServer` - to verify tool registration
- `WebApi` connection - to mock Azure DevOps API calls
- `fetch` - for direct HTTP calls

Key test patterns from existing tests:

```typescript
beforeEach(() => {
  server = { tool: jest.fn() } as unknown as McpServer;
  mockCoreApi = { getProjects: jest.fn() };
  mockConnection = { getCoreApi: jest.fn().mockResolvedValue(mockCoreApi) };
  connectionProvider = jest.fn().mockResolvedValue(mockConnection);
});
```

## Adding a New Tool

1. Identify the appropriate domain file in `src/tools/` (or create a new one)
2. Add tool name constant to the `*_TOOLS` object
3. Implement the tool using `server.tool()` with Zod schema
4. Add tests in corresponding `test/src/tools/` file
5. Run `npm run validate-tools` to ensure naming compliance
6. Update `docs/TOOLSET.md` with documentation
