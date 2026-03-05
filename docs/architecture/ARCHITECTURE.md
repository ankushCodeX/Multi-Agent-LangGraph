# System Architecture

## Overview

The system is a **model-agnostic, loosely coupled multi-agent platform** built on LangGraph (Python) for orchestration and the MCP SDK (TypeScript/Python) for tool integrations. Every component communicates through defined contracts — no component reaches into the internals of another.

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                          TRIGGER LAYER                               │
│          Webhooks (Alerts) │ Slack Events │ REST API │ CLI           │
└───────────────────────────────────┬──────────────────────────────────┘
                                    │ Structured TaskRequest
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         API GATEWAY (FastAPI)                        │
│         Auth │ Rate Limiting │ Request Validation │ Logging          │
└───────────────────────────────────┬──────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      ORCHESTRATOR  (LangGraph)                       │
│                                                                      │
│   ┌─────────────────┐   ┌──────────────────┐   ┌────────────────┐  │
│   │ Intent          │──▶│ Task Decomposer  │──▶│ Agent Router   │  │
│   │ Classifier      │   │                  │   │                │  │
│   └─────────────────┘   └──────────────────┘   └───────┬────────┘  │
│                                                         │            │
│   ┌─────────────────────────────────────────────────────▼──────┐   │
│   │                    LLM ADAPTER (abstraction layer)          │   │
│   │    OpenAI │ Azure OpenAI │ Bedrock │ Ollama │ Any LLM       │   │
│   └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                    │ Dispatches to specialist agent
              ┌─────────────────────┼──────────────────────┐
              ▼                     ▼                       ▼
   ┌─────────────────┐   ┌──────────────────┐   ┌──────────────────┐
   │  DevOps Agent   │   │ Workflow Agent   │   │ Knowledge Agent  │
   │  [P1]           │   │ [P2]             │   │ [P3]             │
   └────────┬────────┘   └────────┬─────────┘   └────────┬─────────┘
            │                     │                       │
            └─────────────────────┴───────────────────────┘
                                  │ Invokes skills
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│                          SKILLS LIBRARY                              │
│   observability/ │ workflow/ │ knowledge/ │ sdlc/                   │
│   Each skill = input contract + output contract + MCP call(s)       │
└───────────────────────────────────┬──────────────────────────────────┘
                                    │ MCP protocol calls
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        MCP SERVER REGISTRY                           │
│                                                                      │
│   ┌──────────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│   │ AppDynamics  │  │  Splunk  │  │PagerDuty │  │     Slack     │  │
│   │ MCP Server   │  │  MCP     │  │  MCP     │  │     MCP       │  │
│   └──────────────┘  └──────────┘  └──────────┘  └───────────────┘  │
│   ┌──────────────┐  ┌──────────┐  ┌──────────┐                     │
│   │     Jira     │  │Confluence│  │  GitHub  │                     │
│   │    MCP       │  │   MCP    │  │   MCP    │                     │
│   └──────────────┘  └──────────┘  └──────────┘                     │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Component Descriptions

### 1. Trigger Layer

Accepts inbound events and normalizes them into a `TaskRequest` object before passing to the gateway.

| Trigger Type | Description |
|---|---|
| Alert Webhook | AppDynamics / Splunk fires HTTP POST on alert threshold |
| Slack Event | Slash command or mention in configured channel |
| REST API | Direct call from CI/CD pipelines or other services |
| CLI | Local developer invocation for testing |

**Output contract — `TaskRequest`:**
```json
{
  "task_id": "uuid",
  "source": "webhook|slack|api|cli",
  "context": { "...raw payload..." },
  "timestamp": "ISO8601",
  "priority": "P1|P2|P3"
}
```

---

### 2. API Gateway (FastAPI)

The security and reliability boundary. All inbound calls pass through here.

**Responsibilities:**
- Bearer token / API key authentication
- Request schema validation
- Rate limiting per caller
- Structured logging of every request
- Routes valid requests to Orchestrator

**Note:** In hybrid deployment, this is the boundary between on-prem and cloud. Claude API calls exit from behind the gateway; internal MCP calls never leave.

---

### 3. Orchestrator (LangGraph)

The brain of the system. Manages the state graph for each task.

**Sub-components:**

**Intent Classifier**
- Takes `TaskRequest.context`
- Returns: `intent_type` (observability / workflow / knowledge / sdlc / unknown)
- Uses: LLM call via LLM Adapter, or rule-based for known alert patterns (faster)

**Task Decomposer**
- Breaks complex requests into ordered sub-tasks
- Example: "Investigate alert AND create incident AND notify team" → 3 sub-tasks

**Agent Router**
- Maps intent + sub-tasks to one or more specialist agents
- Maintains task state across agent handoffs via Redis

---

### 4. LLM Adapter (Model-Agnostic Layer)

The most critical architectural boundary for org independence.

```
┌────────────────────────────────────┐
│          LLMAdapter (abstract)     │
│  + complete(messages, tools) → str │
│  + stream(messages) → Iterator     │
└────────────────────────────────────┘
         ▲              ▲              ▲
         │              │              │
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ OpenAIAdapter│ │AzureOAIAdapt.│ │BedrockAdapter│
│              │ │(org default) │ │              │
└──────────────┘ └──────────────┘ └──────────────┘
```

- No agent, skill, or orchestrator logic ever calls an LLM directly
- LLM provider is configured via environment variable: `LLM_PROVIDER=azure_openai`
- Switching LLM = changing one env var, zero code changes

---

### 5. Specialist Agents

Each agent is a self-contained unit with:
- A system prompt defining its role and constraints
- A declared list of skills it is allowed to invoke
- No direct MCP or LLM access — all via abstractions

| Agent | Priority | Allowed Skills |
|---|---|---|
| DevOps Agent | P1 | alert_triage, log_analysis, metric_correlation, incident_creator, runbook_lookup |
| Workflow Agent | P2 | task_router, approval_chain, status_broadcaster, escalation_handler |
| Knowledge Agent | P3 | confluence_search, doc_summarizer, runbook_generator |
| SDLC Agent | P4 | pr_reviewer, code_context, deploy_checker |

---

### 6. Skills Library

Skills are the atomic units of capability. Each skill:
- Has a strict typed input schema
- Has a strict typed output schema
- Internally calls one or more MCP servers
- Can be tested in complete isolation using mock MCP responses

**Skill file structure:**
```
skills/
  observability/
    alert_triage.py          # Skill implementation
    alert_triage.schema.json # Input/output contract
    alert_triage.test.py     # Unit tests (mock MCP)
  workflow/
    ...
  knowledge/
    ...
  sdlc/
    ...
```

---

### 7. MCP Server Registry

A configuration-driven registry of all available MCP servers.

```yaml
# registry.yaml
servers:
  - name: splunk-mcp
    url: "${SPLUNK_MCP_URL}"          # injected from env
    type: observability
    tools: [search_logs, get_alerts, create_saved_search]
    auth_type: bearer_token
    auth_env_var: SPLUNK_MCP_TOKEN
    deployment: on-prem
    health_check: /health

  - name: pagerduty-mcp
    url: "${PAGERDUTY_MCP_URL}"
    type: incident_management
    tools: [create_incident, update_incident, get_on_call]
    auth_type: api_key
    auth_env_var: PAGERDUTY_API_KEY
    deployment: cloud
    health_check: /health
```

Each MCP server is a separate deployable service — see [MCP_SERVER_GUIDE.md](MCP_SERVER_GUIDE.md).

---

### 8. State & Memory (Redis)

Used for:
- Task state persistence across agent handoffs
- Conversation history within a session
- Deduplication of incoming alerts
- Rate limit counters

Redis is the only stateful dependency in the system. If Redis is unavailable, the system degrades gracefully to stateless mode (no cross-agent handoffs).

---

## Data Flow: Alert Triage (P1 Use Case)

```
AppDynamics Alert
      │
      ▼ POST /webhook
API Gateway (validates, authenticates)
      │
      ▼ TaskRequest{source=webhook, priority=P1}
Orchestrator
  → Intent Classifier:  "observability"
  → Task Decomposer:    [triage_alert, lookup_runbook, create_incident, notify_slack]
  → Agent Router:       DevOps Agent
      │
      ▼
DevOps Agent
  → Skill: alert_triage(alert_payload)
      → MCP: appdynamics-mcp.get_alert_details()
      → MCP: splunk-mcp.search_logs(service, timerange)
      → LLM: summarize findings
  → Skill: runbook_lookup(alert_type)
      → MCP: confluence-mcp.search(alert_type + "runbook")
  → Skill: incident_creator(summary, severity)
      → MCP: pagerduty-mcp.create_incident()
      → MCP: jira-mcp.create_ticket()
  → Skill: status_broadcaster(incident_details)
      → MCP: slack-mcp.post_message(channel=#incidents)
      │
      ▼
Orchestrator aggregates results → TaskResponse → API Gateway → caller
```

---

## Future: Org Agent Provider Integration

When the organization's common Agent Provider platform becomes available, the architecture is already structured for handoff:

- **MCP servers** are independently deployed services — register their URLs in the org platform
- **Skills library** is callable via defined contracts — expose as tools in org platform
- **Agents** are portable — their system prompts and skill lists are configuration, not code
- **Orchestrator** can be replaced by org platform's routing — or kept as a local pre-processor

See [AGENT_PROVIDER_GUIDE.md](../guides/AGENT_PROVIDER_GUIDE.md) for the transition strategy.
