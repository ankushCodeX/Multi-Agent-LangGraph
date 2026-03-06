# Skill: MCP Servers — Base Server Template
**Phase:** 5-A | **Type:** High-level Structure | **Depends on:** 00-B (nexus-config)

---

## Context
The canonical template for every MCP server in the system. Each server is a
standalone TypeScript/Node.js process. Copy this template, rename, and add
tool implementations. Designed to be publishable to the org's common agent
platform in Phase 5 with zero structural changes.

---

## Parameters

| Parameter | Description |
|---|---|
| `{{SERVER_NAME}}` | e.g. `appdynamics-mcp` |
| `{{SERVER_DISPLAY_NAME}}` | e.g. `AppDynamics` |
| `{{TOOL_API_BASE_URL}}` | The tool's API base URL |
| `{{API_KEY_ENV_VAR}}` | Env var for the tool's API key |
| `{{NODE_VERSION}}` | e.g. `18` |
| `{{NEXUS_REGISTRY_URL}}` | Nexus npm registry URL |

---

## Dependencies
- `00-B` .npmrc and tsconfig-base.json exist

---

## Prompt

```
Generate a complete standalone MCP server project at path:
mcp-servers/{{SERVER_NAME}}/

The server connects to {{SERVER_DISPLAY_NAME}} ({{TOOL_API_BASE_URL}}) and exposes
its capabilities as MCP tools. Use @modelcontextprotocol/sdk and TypeScript.

Generate these files:

1. mcp-servers/{{SERVER_NAME}}/package.json
   - name: "@internal/{{SERVER_NAME}}"
   - version: "1.0.0"
   - private: true
   - main: "dist/index.js"
   - scripts: build (tsc), start (node dist/index.js), dev (ts-node src/index.ts),
     test (jest)
   - dependencies: @modelcontextprotocol/sdk, axios, zod, dotenv, winston
   - devDependencies: typescript, ts-node, jest, ts-jest, @types/jest, @types/node
   - registry pointing to {{NEXUS_REGISTRY_URL}}

2. mcp-servers/{{SERVER_NAME}}/tsconfig.json
   - extends: "../../mcp-servers/tsconfig-base.json"

3. mcp-servers/{{SERVER_NAME}}/src/config.ts
   - Reads {{API_KEY_ENV_VAR}} from process.env
   - Reads {{SERVER_NAME|upper|replace("-","_")}}_BASE_URL from env
     (defaults to "{{TOOL_API_BASE_URL}}")
   - Exports: { apiKey: string, baseUrl: string, timeout: number }
   - Throws clear error on startup if apiKey is missing
   - Never exports apiKey directly — use a getter function getApiKey()

4. mcp-servers/{{SERVER_NAME}}/src/client.ts
   - ApiClient class wrapping axios
   - Constructor: (baseUrl: string)
   - Method: async request<T>(method, path, data?, params?) -> Promise<T>
     - Attaches Authorization: Bearer header from getApiKey()
     - Sets timeout from config
     - On 401: throws McpAuthError
     - On 429: throws McpRateLimitError (with retryAfterMs if header present)
     - On 5xx: throws McpServerError
     - All errors extend base McpClientError
   - Separate error classes file: src/errors.ts

5. mcp-servers/{{SERVER_NAME}}/src/tools.ts
   - TOOLS array: MCP tool definitions (name, description, inputSchema using zod)
   - Placeholder tool: "{{SERVER_NAME}}_health_check"
     Description: "Check if {{SERVER_DISPLAY_NAME}} is reachable"
     Input: {} (no params)
     Returns: { status: "ok" | "error", latency_ms: number }
   - exportToolDefinitions() -> Tool[] for use in index.ts

6. mcp-servers/{{SERVER_NAME}}/src/handlers.ts
   - ToolHandlers: Record<string, (args: unknown) => Promise<unknown>>
   - Handler for "{{SERVER_NAME}}_health_check":
     Makes GET to /health or equivalent, measures latency, returns result
   - handleToolCall(toolName: string, args: unknown) -> Promise<CallToolResult>
     Looks up handler, runs it, wraps in MCP CallToolResult format
     On error: returns isError=true result (never throws — MCP protocol requirement)

7. mcp-servers/{{SERVER_NAME}}/src/index.ts
   - Creates McpServer instance from @modelcontextprotocol/sdk
   - Registers all tools from tools.ts
   - Implements CallToolRequestSchema handler using handleToolCall
   - Connects via StdioServerTransport (default) or HTTP based on
     MCP_TRANSPORT env var ("stdio" | "http")
   - Startup logging: server name, version, transport type
   - Graceful shutdown on SIGTERM/SIGINT

8. mcp-servers/{{SERVER_NAME}}/src/logger.ts
   - Winston logger, JSON format (Splunk-compatible)
   - Log level from LOG_LEVEL env var (default: "info")
   - Never log API keys or auth headers

9. mcp-servers/{{SERVER_NAME}}/.env.example
   {{API_KEY_ENV_VAR}}=your_key_here
   {{SERVER_NAME|upper|replace("-","_")}}_BASE_URL={{TOOL_API_BASE_URL}}
   LOG_LEVEL=info
   MCP_TRANSPORT=stdio

10. mcp-servers/{{SERVER_NAME}}/tests/server.test.ts
    - Jest test for health_check tool handler
    - Mock axios with jest.mock
    - Test: successful health check returns status "ok"
    - Test: API error returns isError=true result (not thrown exception)
    - Test: missing API key at startup throws clear error

Rules:
- All files complete and production-ready
- No circular imports
- Errors never propagate out of MCP handlers (return error result instead)
- API keys never in logs, never in error messages returned to MCP client
```

---

## Expected Output
10 files, fully runnable MCP server for {{SERVER_DISPLAY_NAME}}

## Verification
```bash
cd mcp-servers/{{SERVER_NAME}}
npm install
npm run build
# Should compile with no TypeScript errors

npm test
# Should pass 3 tests

# Smoke test:
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | node dist/index.js
# Should return JSON with health_check tool listed
```
