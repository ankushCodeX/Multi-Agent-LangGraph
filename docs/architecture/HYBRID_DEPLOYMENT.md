# Hybrid Deployment Architecture

## Overview

The system operates across two zones: **on-premises (private cloud)** where sensitive tooling lives, and **cloud (SaaS)** where third-party services live. The FastAPI gateway is the sole boundary between the two zones.

---

## Deployment Zones

```
╔══════════════════════════════════════════════════════╗
║              ON-PREMISES / PRIVATE CLOUD             ║
║                                                      ║
║   ┌──────────────────────────────────────────────┐  ║
║   │         FastAPI Gateway (Security Boundary)  │  ║
║   │   Auth │ Rate Limit │ Logging │ Secret Mgmt  │  ║
║   └─────────────────────┬────────────────────────┘  ║
║                         │                            ║
║   ┌─────────────────────▼────────────────────────┐  ║
║   │        Orchestrator (LangGraph)              │  ║
║   │        + Agent Pool                          │  ║
║   │        + Skills Library                      │  ║
║   └──────────────────────────────────────────────┘  ║
║                                                      ║
║   ┌──────────────┐  ┌──────────┐  ┌─────────────┐  ║
║   │AppDynamics   │  │  Splunk  │  │ Confluence  │  ║
║   │  MCP Server  │  │   MCP    │  │    MCP      │  ║
║   └──────────────┘  └──────────┘  └─────────────┘  ║
║                                                      ║
║   ┌──────────────┐  ┌──────────┐                   ║
║   │  Redis       │  │  Nexus   │                   ║
║   │  (State)     │  │  (Pkgs)  │                   ║
║   └──────────────┘  └──────────┘                   ║
╚══════════════════════════════════════════════════════╝
                         │
                  HTTPS (outbound only)
                  from Gateway / LLM Adapter
                         │
╔══════════════════════════════════════════════════════╗
║                   CLOUD / SaaS                       ║
║                                                      ║
║   ┌──────────────┐  ┌──────────┐  ┌─────────────┐  ║
║   │  LLM API     │  │PagerDuty │  │    Slack    │  ║
║   │  (Org Model  │  │   MCP    │  │    MCP      │  ║
║   │  Directory)  │  │          │  │             │  ║
║   └──────────────┘  └──────────┘  └─────────────┘  ║
║                                                      ║
║   ┌──────────────┐  ┌──────────┐                   ║
║   │  Jira Cloud  │  │  GitHub  │                   ║
║   │    MCP       │  │   MCP    │                   ║
║   └──────────────┘  └──────────┘                   ║
╚══════════════════════════════════════════════════════╝
```

---

## Zone Classification Per Component

| Component | Zone | Reason |
|---|---|---|
| FastAPI Gateway | On-prem | Security boundary, all traffic passes through |
| LangGraph Orchestrator | On-prem | Processes internal alert data — must not leave network |
| Skills Library | On-prem | Invoked within orchestrator process |
| Redis | On-prem | Stores task context and sensitive session state |
| AppDynamics MCP | On-prem | Internal APM — data must not leave network |
| Splunk MCP | On-prem | Internal logs — data must not leave network |
| Confluence MCP | On-prem | Internal knowledge base |
| LLM API | Cloud (outbound) | Org Model Directory endpoint — only summarized/anonymized prompts sent |
| PagerDuty MCP | Cloud | SaaS incident management |
| Slack MCP | Cloud | SaaS messaging |
| Jira MCP | Cloud or On-prem | Depends on org Jira deployment |
| GitHub MCP | Cloud or On-prem | Depends on org GitHub deployment |

---

## Data Handling Rules

### What NEVER leaves the on-prem boundary:
- Raw log data from Splunk
- Raw metric data from AppDynamics
- Internal Confluence document contents
- Authentication tokens and secrets
- Full alert payloads

### What CAN leave (to LLM API):
- Summarized, anonymized descriptions of issues
- Task instructions (e.g. "Summarize this alert pattern")
- Structured outputs for reasoning (with PII stripped)

> **Implementation note:** The LLM Adapter layer is responsible for data sanitization before sending to external LLM. A `sanitize_for_llm(text)` utility will be provided in `shared/utils/`.

---

## Secret Management

Secrets are never hardcoded or committed to the repository.

| Secret Type | Storage | Access Pattern |
|---|---|---|
| LLM API Key | Org vault / K8s secret | Injected as env var at pod startup |
| MCP Bearer Tokens | Org vault / K8s secret | Injected as env var at pod startup |
| Redis password | Org vault / K8s secret | Injected as env var at pod startup |
| Internal tool API keys | Org vault / K8s secret | Injected as env var at pod startup |

**Environment variable naming convention:**
```bash
# MCP server auth tokens
{SERVER_NAME_UPPER}_MCP_TOKEN=...
# Example:
SPLUNK_MCP_TOKEN=...
APPDYNAMICS_MCP_TOKEN=...

# LLM
LLM_API_KEY=...
LLM_API_BASE=...

# Redis
REDIS_URL=redis://redis-host:6379
REDIS_PASSWORD=...
```

---

## Network Requirements

Ports and endpoints that need to be opened. Share this list with your network/infra team.

### Inbound (to on-prem Gateway):
| Source | Protocol | Port | Purpose |
|---|---|---|---|
| AppDynamics Alert Engine | HTTPS | 443 | Alert webhooks |
| Splunk Alert Actions | HTTPS | 443 | Alert webhooks |
| Slack | HTTPS | 443 | Slack event subscriptions |
| Internal CI/CD | HTTPS | 443 | API calls |

### Outbound (from on-prem):
| Destination | Protocol | Port | Purpose |
|---|---|---|---|
| Org LLM API endpoint | HTTPS | 443 | LLM completions |
| PagerDuty API | HTTPS | 443 | Incident management |
| Slack API | HTTPS | 443 | Notifications |
| Jira Cloud (if cloud) | HTTPS | 443 | Ticket management |
| GitHub (if cloud) | HTTPS | 443 | Code operations |

### Internal (on-prem to on-prem):
| From | To | Protocol | Port |
|---|---|---|---|
| Orchestrator | Splunk MCP Server | HTTP | 8001 |
| Orchestrator | AppDynamics MCP | HTTP | 8002 |
| Orchestrator | Confluence MCP | HTTP | 8003 |
| All components | Redis | TCP | 6379 |
| All MCP Servers | Splunk/AppD/Confluence | HTTPS | 443 |

---

## Container / Deployment Units

Each component is a separate deployable unit. This enables independent scaling and deployment.

```yaml
# deployment units (each is a separate container/pod)
services:
  api-gateway:        # FastAPI gateway
  orchestrator:       # LangGraph + agents + skills
  mcp-splunk:         # Splunk MCP server
  mcp-appdynamics:    # AppDynamics MCP server
  mcp-confluence:     # Confluence MCP server
  redis:              # State store
```

Cloud MCP servers (PagerDuty, Slack, Jira, GitHub) are either:
- Deployed as lightweight cloud functions / containers in org's cloud account
- Or using community MCP servers if available and approved by security team

---

## Graceful Degradation

| Failure | System Behavior |
|---|---|
| Redis unavailable | Continue in stateless mode — no cross-agent handoffs, no deduplication |
| One MCP server down | Skill returns partial result — agent continues with available data |
| LLM API unavailable | Orchestrator returns error with raw data collected so far |
| Cloud MCP (Slack/PD) unavailable | Skip notification step — core triage still completes |
| Gateway restart | In-flight requests fail cleanly — callers receive 503 with retry guidance |
