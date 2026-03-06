# Skill: MCP Server — Splunk
**Phase:** 5-C | **Type:** Granular Logic | **Depends on:** 05-A (mcp-server-template)

---

## Context
Splunk MCP server. Run mcp-server-template skill with SERVER_NAME=splunk-mcp first,
then extend with these Splunk-specific tools.

---

## Parameters

| Parameter | Description |
|---|---|
| `{{SPLUNK_BASE_URL}}` | Splunk REST API base URL e.g. `https://splunk.internal.org:8089` |
| `{{API_KEY_ENV_VAR}}` | e.g. `SPLUNK_TOKEN` (Splunk uses bearer token auth) |

---

## Prompt

```
Extend mcp-servers/splunk-mcp/ (template already applied) with Splunk tools.
UPDATE src/tools.ts and src/handlers.ts:

Tool 1: "search_logs"
  Input: query (str, SPL query), time_range_minutes (number, default 60),
         max_results (number, default 100)
  Output: { events: Array<{_time, _raw, source, host, log_level}>,
            total_count, search_job_id }
  Handler: POST /services/search/jobs (create job), poll until done,
           GET /services/search/jobs/{id}/results

Tool 2: "get_log_summary"
  Input: service_name (str), time_range_minutes (number, default 30),
         error_only (boolean, default false)
  Output: { error_count, warn_count, info_count, top_errors: string[],
            sample_log_lines: string[] }
  Handler: Build SPL query from params, run search_logs internally

Tool 3: "create_saved_search_alert"
  Input: name (str), search_query (str), cron_schedule (str),
         alert_threshold (number)
  Output: { alert_id, name, status: "created" | "already_exists" }

Tool 4: "get_dashboard_data"
  Input: dashboard_name (str), time_range_minutes (number, default 60)
  Output: { panels: Array<{title, data_type, values}> }

Splunk-specific handler notes:
- Auth: Authorization: Bearer {{API_KEY_ENV_VAR value}}
- Content-Type: application/x-www-form-urlencoded for search job creation
- Poll search job with GET /services/search/jobs/{id} until dispatchState="DONE"
- Max poll attempts: 30, interval: 2s (handle long-running searches)
- SPL queries must be URL-encoded in request body
- Splunk returns XML by default — always add output_mode=json param
```

---

## Verification
```bash
cd mcp-servers/splunk-mcp && npm run build && npm test
```

---
---

# Skill: MCP Server — Jira
**Phase:** 5-E | **Type:** Granular Logic | **Depends on:** 05-A

---

## Parameters

| Parameter | Description |
|---|---|
| `{{JIRA_BASE_URL}}` | Jira instance URL |
| `{{API_KEY_ENV_VAR}}` | e.g. `JIRA_API_TOKEN` |

---

## Prompt

```
Extend mcp-servers/jira-mcp/ (template already applied) with Jira tools.

Tool 1: "create_incident_ticket"
  Input: summary (str), description (str), priority (enum: Low/Medium/High/Critical),
         project_key (str), issue_type (str, default "Incident"),
         labels (string[], default [])
  Output: { ticket_id, ticket_url, status }
  Handler: POST {{JIRA_BASE_URL}}/rest/api/3/issue
  Auth: Basic base64(email:api_token) — Jira Cloud uses email+token not username+token

Tool 2: "get_ticket"
  Input: ticket_id (str)
  Output: { id, summary, status, priority, assignee, created_at, updated_at,
            description, comments_count }

Tool 3: "update_ticket_status"
  Input: ticket_id (str), transition_name (str e.g. "In Progress", "Done")
  Output: { success, new_status }
  Handler: GET /issue/{id}/transitions to find transition_id, then POST /transitions

Tool 4: "add_comment"
  Input: ticket_id (str), comment (str), author_display (str, default "MAS Bot")
  Output: { comment_id, created_at }

Tool 5: "search_tickets"
  Input: jql (str), max_results (number, default 20)
  Output: { issues: Array<{id, summary, status, priority}>, total }
  Handler: POST {{JIRA_BASE_URL}}/rest/api/3/search with JQL

Notes:
- Jira API v3 returns rich text (Atlassian Document Format) — convert to plain text
- Rate limit: Jira Cloud is 100 req/min — add rate limit tracking
```

---

## Verification
```bash
cd mcp-servers/jira-mcp && npm run build && npm test
```

---
---

# Skill: MCP Server — Confluence
**Phase:** 5-F | **Type:** Granular Logic | **Depends on:** 05-A

---

## Parameters

| Parameter | Description |
|---|---|
| `{{CONFLUENCE_BASE_URL}}` | Confluence instance URL |
| `{{API_KEY_ENV_VAR}}` | e.g. `CONFLUENCE_API_TOKEN` |

---

## Prompt

```
Extend mcp-servers/confluence-mcp/ (template already applied) with Confluence tools.

Tool 1: "search_content"
  Input: query (str), space_key (str | optional), content_type
         (enum: "page"|"blogpost"|"all", default "all"), max_results (number, default 10)
  Output: { results: Array<{id, title, url, excerpt, last_modified, space}>, total }
  Handler: GET {{CONFLUENCE_BASE_URL}}/rest/api/content/search?cql=...
  Build CQL: text ~ "{query}" AND type = "{type}" AND space = "{space_key}"

Tool 2: "get_page_content"
  Input: page_id (str), include_body (boolean, default true)
  Output: { id, title, body_plain_text, url, version, last_modified, author }
  Handler: GET /rest/api/content/{id}?expand=body.storage,version,history
  Convert storage format (HTML-like) to plain text

Tool 3: "get_runbook"
  Input: category (str), keywords (string[])
  Output: { found: boolean, title, content, url, last_modified }
  Handler: search_content internally with query built from category+keywords
  Look in "RUNBOOKS" or "OPS" space first, then global
  Return the most recently modified matching page

Tool 4: "create_incident_page"
  Input: title (str), space_key (str), incident_summary (str),
         parent_page_id (str | optional)
  Output: { page_id, url }
  Handler: POST /rest/api/content — creates page in given space
  Use HTML template for incident documentation format

Notes:
- Confluence REST API v1 (not v2) for broader compatibility
- Storage format is XHTML-like — strip tags for plain text output
- Auth: Basic or Bearer depending on Confluence version (Cloud vs Server)
```

---

## Verification
```bash
cd mcp-servers/confluence-mcp && npm run build && npm test
```

---
---

# Skill: MCP Server — Slack
**Phase:** 5-G | **Type:** Granular Logic | **Depends on:** 05-A

---

## Parameters

| Parameter | Description |
|---|---|
| `{{API_KEY_ENV_VAR}}` | e.g. `SLACK_BOT_TOKEN` (xoxb- token) |

---

## Prompt

```
Extend mcp-servers/slack-mcp/ (template already applied) with Slack tools.
Slack base URL is always https://slack.com/api — hardcode this.

Tool 1: "send_message"
  Input: channel (str, e.g. "#incidents"), text (str),
         thread_ts (str | optional, for threading)
  Output: { ts, channel, permalink }
  Handler: POST https://slack.com/api/chat.postMessage

Tool 2: "send_rich_message"
  Input: channel (str), blocks (any[] — Slack Block Kit JSON),
         fallback_text (str)
  Output: { ts, channel }
  Handler: POST chat.postMessage with blocks param

Tool 3: "create_incident_notification"
  Input: channel (str), severity (enum: low/medium/high/critical),
         summary (str), ticket_url (str | optional), runbook_url (str | optional)
  Output: { ts, channel }
  Handler: Builds a formatted Block Kit message from inputs
  Use color-coded header: 🔴 CRITICAL, 🟠 HIGH, 🟡 MEDIUM, 🟢 LOW
  Include buttons for ticket_url and runbook_url if provided

Tool 4: "get_channel_messages"
  Input: channel_id (str), limit (number, default 20),
         oldest_ts (str | optional)
  Output: { messages: Array<{ts, user, text, reactions}>, has_more }
  Handler: GET conversations.history

Tool 5: "add_reaction"
  Input: channel (str), message_ts (str), reaction (str, e.g. "white_check_mark")
  Output: { success }

Notes:
- All Slack API calls use Authorization: Bearer {SLACK_BOT_TOKEN}
- Slack rate limits: tier 3 = 50 req/min for most methods
- Error format: { ok: false, error: "channel_not_found" } — check ok field
```

---

## Verification
```bash
cd mcp-servers/slack-mcp && npm run build && npm test
```
