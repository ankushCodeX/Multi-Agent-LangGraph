# MCP Server Build Guide

## What is an MCP Server (in this project)?

An MCP (Model Context Protocol) server is a lightweight, standalone HTTP service that wraps one internal tool (Splunk, AppDynamics, Confluence, etc.) and exposes its capabilities as typed, callable "tools" over the MCP protocol. The Skills Library calls these tools via the `MCPClient` wrapper.

Each MCP server is:
- **Independently deployable** — its own container, its own process
- **Independently testable** — unit tests mock all HTTP calls to the underlying tool
- **Self-describing** — `/describe` endpoint lists available tools and their schemas
- **Future publishable** — once an org Agent Provider platform exists, these servers can be registered there directly

---

## Directory Structure (per MCP server)

```
src/mcp-servers/{tool-name}/
├── src/
│   ├── index.ts              ← Entry point — registers tools, starts server
│   ├── client.ts             ← HTTP client wrapper for the underlying tool API
│   └── tools/
│       ├── tool_one.ts       ← One file per tool
│       └── tool_two.ts
├── tests/
│   ├── client.test.ts        ← Unit tests for API client
│   └── tools/
│       ├── tool_one.test.ts  ← Unit tests per tool (all HTTP mocked)
│       └── tool_two.test.ts
├── Dockerfile
├── package.json
├── tsconfig.json
└── README.md                 ← Tool-specific auth and setup instructions
```

---

## Step-by-Step: Building a New MCP Server

### Step 1 — Confirm Nexus availability

Before writing any code, check `DEPENDENCIES.md` and confirm these Node.js packages are in Nexus:
- `@modelcontextprotocol/sdk`
- `zod`
- `typescript`, `ts-node`, `@types/node`
- `axios` (for HTTP calls to the tool API)
- `jest`, `nock` (for tests)

### Step 2 — Create the directory

```bash
mkdir -p src/mcp-servers/{tool-name}/src/tools
mkdir -p src/mcp-servers/{tool-name}/tests/tools
```

### Step 3 — package.json

```json
{
  "name": "{tool-name}-mcp-server",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "ts-node src/index.ts",
    "test": "jest",
    "test:coverage": "jest --coverage"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "0.6.0",
    "axios": "1.6.7",
    "dotenv": "16.4.1",
    "zod": "3.22.4"
  },
  "devDependencies": {
    "@types/node": "20.11.5",
    "jest": "29.7.0",
    "@types/jest": "29.5.11",
    "nock": "13.5.0",
    "ts-jest": "29.1.2",
    "typescript": "5.3.3"
  }
}
```

### Step 4 — Tool implementation

Each tool in `src/tools/{tool_name}.ts` follows this pattern:

```typescript
import { z } from "zod";
import { ToolClient } from "../client";

// Input schema — validated by MCP SDK automatically
export const SearchLogsInputSchema = z.object({
  query: z.string().describe("Splunk SPL query string"),
  service_name: z.string().describe("Service to filter logs for"),
  time_range_minutes: z.number().default(30).describe("How far back to search"),
});

export type SearchLogsInput = z.infer<typeof SearchLogsInputSchema>;

// Output is always a plain object — MCP SDK serializes it
export interface SearchLogsOutput {
  total_events: number;
  events: Array<{ timestamp: string; message: string; severity: string }>;
  search_url: string;
}

export async function searchLogs(
  input: SearchLogsInput,
  client: ToolClient
): Promise<SearchLogsOutput> {
  const results = await client.search(input.query, input.service_name, input.time_range_minutes);
  return {
    total_events: results.count,
    events: results.events.map((e) => ({
      timestamp: e._time,
      message: e._raw,
      severity: e.severity ?? "unknown",
    })),
    search_url: results.search_url,
  };
}
```

### Step 5 — Entry point (index.ts)

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { CallToolRequestSchema, ListToolsRequestSchema } from "@modelcontextprotocol/sdk/types.js";
import { ToolClient } from "./client";
import { SearchLogsInputSchema, searchLogs } from "./tools/search_logs";

const client = new ToolClient({
  baseUrl: process.env.TOOL_BASE_URL!,
  token: process.env.TOOL_API_TOKEN!,
});

const server = new Server(
  { name: "{tool-name}-mcp", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// List available tools
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "search_logs",
      description: "Search logs by query and service name",
      inputSchema: {
        type: "object",
        properties: {
          query: { type: "string" },
          service_name: { type: "string" },
          time_range_minutes: { type: "number" },
        },
        required: ["query", "service_name"],
      },
    },
  ],
}));

// Handle tool calls
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  switch (request.params.name) {
    case "search_logs": {
      const input = SearchLogsInputSchema.parse(request.params.arguments);
      const result = await searchLogs(input, client);
      return { content: [{ type: "text", text: JSON.stringify(result) }] };
    }
    default:
      throw new Error(`Unknown tool: ${request.params.name}`);
  }
});

// Health and describe endpoints (add via HTTP wrapper in production)
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
}

main().catch(console.error);
```

### Step 6 — Write tests (all HTTP mocked)

```typescript
// tests/tools/search_logs.test.ts
import nock from "nock";
import { searchLogs } from "../../src/tools/search_logs";
import { ToolClient } from "../../src/client";

describe("searchLogs", () => {
  const client = new ToolClient({ baseUrl: "https://splunk-test", token: "test-token" });

  beforeEach(() => nock.cleanAll());
  afterAll(() => nock.restore());

  it("returns formatted events on success", async () => {
    nock("https://splunk-test")
      .get("/services/search/jobs/export")
      .query(true)
      .reply(200, {
        count: 2,
        events: [
          { _time: "2026-01-01T00:00:00", _raw: "error in service", severity: "error" },
        ],
        search_url: "https://splunk/search/123",
      });

    const result = await searchLogs(
      { query: "index=main error", service_name: "payment-service", time_range_minutes: 30 },
      client
    );

    expect(result.total_events).toBe(2);
    expect(result.events[0].message).toBe("error in service");
  });

  it("throws on non-200 response", async () => {
    nock("https://splunk-test").get("/services/search/jobs/export").query(true).reply(401);
    await expect(
      searchLogs({ query: "test", service_name: "svc", time_range_minutes: 5 }, client)
    ).rejects.toThrow();
  });
});
```

### Step 7 — Dockerfile

```dockerfile
FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
# Install from internal Nexus — configure .npmrc before building
RUN npm ci --registry=${NEXUS_NPM_URL}
COPY . .
RUN npm run build

EXPOSE 3000
ENV NODE_ENV=production

HEALTHCHECK --interval=30s --timeout=5s CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
```

### Step 8 — Register in registry.yaml

```yaml
# src/shared/registry/registry.yaml
servers:
  - name: {tool-name}-mcp
    url: "${TOOL_MCP_URL}"
    type: observability          # or: workflow | knowledge | sdlc
    tools: [search_logs]
    auth_type: bearer_token
    auth_env_var: TOOL_MCP_TOKEN
    deployment: on-prem          # or: cloud
    health_check: /health
```

---

## Publishing to Org Agent Provider Platform (Future)

When the org Agent Provider platform becomes available:

1. Ensure your MCP server's `/describe` endpoint returns valid JSON per the `COMPONENT_CONTRACTS.md` self-description contract
2. Register the server URL in the org platform's MCP registry
3. The platform will call `/describe` to discover available tools automatically
4. No code changes required to the MCP server itself

---

## MCP Server Checklist

Before marking a MCP server as complete:

- [ ] All tools have Zod input schemas
- [ ] All tools have typed output interfaces
- [ ] `client.ts` wraps all HTTP calls — no raw `axios` calls in tool files
- [ ] Unit tests cover: success path, auth failure, network timeout, malformed response
- [ ] `nock.cleanAll()` called in `beforeEach` — no test pollution
- [ ] Dockerfile builds successfully
- [ ] `/health` endpoint returns `200 OK`
- [ ] `/describe` endpoint returns valid self-description JSON
- [ ] Registered in `registry.yaml`
- [ ] `README.md` documents: required env vars, auth setup, tool descriptions
- [ ] No hardcoded URLs, tokens, or credentials anywhere
