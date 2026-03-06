# Skill: MCP Server — AppDynamics
**Phase:** 5-B | **Type:** Granular Logic | **Depends on:** 05-A (mcp-server-template)

---

## Context
Extends the base MCP server template with AppDynamics-specific tools.
These tools are called by the DevOps agent's AlertTriageSkill and MetricCorrelationSkill.
Run the mcp-server-template skill first with SERVER_NAME=appdynamics-mcp, then
run this skill to add the AppDynamics tool implementations.

---

## Parameters

| Parameter | Description |
|---|---|
| `{{APPDYNAMICS_BASE_URL}}` | AppDynamics controller URL |
| `{{API_KEY_ENV_VAR}}` | e.g. `APPDYNAMICS_API_KEY` |

---

## Dependencies
- `05-A` template applied with SERVER_NAME=appdynamics-mcp

---

## Prompt

```
Extend the MCP server at mcp-servers/appdynamics-mcp/ with AppDynamics-specific tools.
The base structure already exists. Add/replace these files:

UPDATE src/tools.ts — add these tool definitions after health_check:

Tool 1: "get_alert_context"
  Description: "Fetch context for an alert from AppDynamics including app, tier, node info"
  Input schema (zod):
    alert_text: z.string().describe("Raw alert text to look up context for")
    time_window_minutes: z.number().default(30)
  Output: { app_name, tier, node, recent_events: string[], health_rule_name }

Tool 2: "get_metric_data"
  Description: "Fetch metric time-series data for a given application and metric path"
  Input schema:
    app_name: z.string()
    metric_path: z.string().describe("e.g. 'Overall Application Performance|Calls per Minute'")
    duration_minutes: z.number().default(60)
  Output: { metric_path, data_points: Array<{timestamp: number, value: number}>, baseline_value }

Tool 3: "list_health_rule_violations"
  Description: "Get active health rule violations for an application"
  Input schema:
    app_name: z.string()
    severity_filter: z.enum(["ALL","WARNING","CRITICAL"]).default("ALL")
  Output: { violations: Array<{id, name, severity, start_time, affected_entity}> }

Tool 4: "get_business_transaction_errors"
  Description: "Get error rate for business transactions in last N minutes"
  Input schema:
    app_name: z.string()
    duration_minutes: z.number().default(15)
    threshold_percent: z.number().default(5.0)
  Output: { transactions: Array<{name, error_rate_percent, call_count, above_threshold: bool}> }

UPDATE src/handlers.ts — implement handlers for each tool:

get_alert_context handler:
  GET {{APPDYNAMICS_BASE_URL}}/controller/rest/applications?output=JSON
  Filter by alert_text keyword matching, fetch application details
  GET /controller/rest/applications/{app_id}/tiers?output=JSON for tier info
  Return structured context

get_metric_data handler:
  GET {{APPDYNAMICS_BASE_URL}}/controller/rest/applications/{app_id}/metric-data
  Query params: metric-path, time-range-type=BEFORE_NOW, duration-in-mins={duration}
  Parse metric data array into {timestamp, value} pairs

list_health_rule_violations handler:
  GET {{APPDYNAMICS_BASE_URL}}/controller/rest/applications/{app_id}/problems/healthrule-violations
  Filter by severity_filter param

get_business_transaction_errors handler:
  GET {{APPDYNAMICS_BASE_URL}}/controller/rest/applications/{app_id}/business-transactions
  Calculate error rates

For ALL handlers:
  - API uses Basic auth (base64 of account@username:password) OR Bearer token
  - Use getApiKey() for the token
  - Parse AppDynamics-specific error responses (returns HTML on some errors — handle gracefully)
  - Return McpCallToolResult with isError=false on success, isError=true on API errors
  - Log tool_name and app_name but NOT the full response (may be large)

UPDATE tests/server.test.ts — add tests:
  - Mock all 4 AppDynamics API endpoints with axios mock
  - Test each tool handler returns correctly shaped output
  - Test AppDynamics HTML error response is handled gracefully (not thrown)
```

---

## Expected Output
- Updated `src/tools.ts` with 5 tools (health_check + 4 new)
- Updated `src/handlers.ts` with 4 new handlers
- Updated `tests/server.test.ts` with 8+ tests

## Verification
```bash
cd mcp-servers/appdynamics-mcp
npm run build && npm test
# Expected: all tests pass, no TS errors
```
