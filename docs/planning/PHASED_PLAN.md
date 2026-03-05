# Phased Delivery Plan

## Guiding Principles for Phasing

1. **Infrastructure before features** — Shared contracts, LLM adapter, and MCP client must exist before any skill is written
2. **P1 use case end-to-end first** — One complete working flow is worth more than partial coverage of all flows
3. **Test harness built in Phase 1** — All subsequent phases inherit working test patterns
4. **Nexus validation gate** — Every new dependency must be confirmed in Nexus before the phase begins

---

## Phase Overview

```
Phase 0 (Foundation)        ██████░░░░░░░░░░░░░░░░░░░░░░  1-2 weeks
Phase 1 (Observability P1)  ██████░░░░░░░░░░░░░░░░░░░░░░  3 weeks
Phase 2 (Workflow P2)       ██████░░░░░░░░░░░░░░░░░░░░░░  2-3 weeks
Phase 3 (Knowledge P3)      ██████░░░░░░░░░░░░░░░░░░░░░░  2 weeks
Phase 4 (SDLC P4)           ██████░░░░░░░░░░░░░░░░░░░░░░  2-3 weeks
Phase 5 (Hardening)         ██████░░░░░░░░░░░░░░░░░░░░░░  2 weeks
```

Total estimated timeline: **12–15 weeks** (from environment setup to production-ready)

---

## Phase 0 — Foundation & Environment Setup

**Duration:** 1–2 weeks  
**Goal:** All shared infrastructure exists and all dependencies are confirmed available

### Deliverables

- [ ] Nexus dependency audit complete (all packages from [DEPENDENCIES.md](DEPENDENCIES.md) verified)
- [ ] Repository structure created and agreed upon
- [ ] Development environment setup guide written
- [ ] `shared/` module scaffolded:
  - `contracts/` — All Pydantic models (TaskRequest, TaskResponse, AgentInput, etc.)
  - `llm_adapter/` — Abstract base + one concrete adapter (org's approved LLM)
  - `mcp_client/` — MCPClient wrapper with logging and timeout handling
  - `registry/` — MCP server registry loader from `registry.yaml`
  - `utils/` — Logging, sanitize_for_llm(), config loader
- [ ] Redis connectivity confirmed in dev environment
- [ ] CI skeleton (lint, type-check, unit test runner) — no external network needed
- [ ] Mock MCP server created for testing (returns fixture data)
- [ ] Mock LLM adapter created for testing (returns fixture responses)

### Exit Criteria
- `shared/` module passes all unit tests with 100% of tests using mocks
- A developer can run `pytest tests/unit/` on a locked-down machine with no internet

---

## Phase 1 — Observability (P1): Alert Triage End-to-End

**Duration:** 3 weeks  
**Goal:** Complete working flow: alert webhook → triage → incident created → Slack notified

### Week 1 — MCP Servers (Splunk + AppDynamics)

- [ ] `mcp-servers/splunk/` — Implement and test independently
  - Tools: `search_logs`, `get_alert_details`, `list_saved_searches`
  - Unit tests with mock Splunk HTTP responses
  - `/health` and `/describe` endpoints
- [ ] `mcp-servers/appdynamics/` — Implement and test independently
  - Tools: `get_alert_details`, `get_metric_data`, `list_affected_services`
  - Unit tests with mock AppDynamics HTTP responses
- [ ] Deploy both MCP servers to dev on-prem environment
- [ ] Manual smoke test against real Splunk/AppDynamics instances

### Week 2 — Skills (Observability Domain)

- [ ] `skills/observability/alert_triage` — alert_triage skill + tests
- [ ] `skills/observability/log_analysis` — log_analysis skill + tests
- [ ] `skills/observability/runbook_lookup` — runbook_lookup skill + tests (uses Confluence — stub for now)
- [ ] `skills/observability/metric_correlation` — metric_correlation skill + tests

### Week 3 — DevOps Agent + Orchestrator + Gateway

- [ ] `agents/devops_agent.py` — agent with system prompt + skill invocation logic
- [ ] `orchestrator/intent_classifier.py` — rule-based classification for P1 alert patterns
- [ ] `orchestrator/task_decomposer.py` — decompose alert tasks into sub-tasks
- [ ] `orchestrator/agent_router.py` — route to DevOps Agent
- [ ] `api_gateway/` — FastAPI app, `/webhook` endpoint, auth middleware
- [ ] `mcp-servers/pagerduty/` — incident_creator MCP server
- [ ] `mcp-servers/slack/` — status_broadcaster MCP server
- [ ] End-to-end integration test: simulated alert → check PagerDuty incident created

### Exit Criteria
- Send a test alert webhook → incident auto-created in PagerDuty → message posted to Slack
- All unit tests pass
- Integration test passes with mock external services

---

## Phase 2 — Cross-Team Workflow (P2)

**Duration:** 2–3 weeks  
**Goal:** Task routing, approvals, and escalation flows working

### Deliverables

- [ ] `mcp-servers/jira/` — MCP server for Jira
  - Tools: `create_ticket`, `update_ticket`, `get_ticket`, `assign_ticket`
- [ ] `skills/workflow/task_router` — route tasks to correct team/queue
- [ ] `skills/workflow/approval_chain` — trigger and track approval flows
- [ ] `skills/workflow/escalation_handler` — auto-escalate stale items
- [ ] `skills/workflow/status_broadcaster` — cross-channel status updates (reuses Slack MCP)
- [ ] `agents/workflow_agent.py` — workflow specialist agent
- [ ] Orchestrator updated: LLM-based intent classification for non-alert tasks
- [ ] End-to-end test: incoming workflow request → Jira ticket → Slack approval → escalation if stale

### Exit Criteria
- Workflow tasks correctly routed to Jira queues
- Approval flow triggers and escalation fires after configurable timeout
- All tests pass

---

## Phase 3 — Knowledge Retrieval (P3)

**Duration:** 2 weeks  
**Goal:** Confluence search, document summarization, runbook generation

### Deliverables

- [ ] `mcp-servers/confluence/` — MCP server for Confluence
  - Tools: `search_pages`, `get_page_content`, `create_page`, `update_page`
- [ ] `skills/knowledge/confluence_search` — semantic search across spaces
- [ ] `skills/knowledge/doc_summarizer` — fetch and summarize any doc
- [ ] `skills/knowledge/runbook_generator` — generate runbook from incident history
- [ ] `agents/knowledge_agent.py` — knowledge specialist agent
- [ ] Phase 1 `runbook_lookup` skill updated — replace Confluence stub with real MCP
- [ ] End-to-end test: question → Confluence search → summary returned

### Exit Criteria
- Alert triage flow now returns actual runbook content (Phase 1 improvement)
- Knowledge agent can answer questions from Confluence content
- All tests pass

---

## Phase 4 — SDLC Automation (P4)

**Duration:** 2–3 weeks  
**Goal:** PR review, code context, deploy status skills

### Deliverables

- [ ] `mcp-servers/github/` — MCP server for GitHub
  - Tools: `get_pr`, `list_pr_files`, `post_pr_comment`, `get_workflow_status`
- [ ] `skills/sdlc/pr_reviewer` — review PR diff, post structured comment
- [ ] `skills/sdlc/code_context` — explain code change to non-developers
- [ ] `skills/sdlc/deploy_checker` — check pipeline/deployment status
- [ ] `agents/sdlc_agent.py` — SDLC specialist agent
- [ ] End-to-end test: PR opened → review comment posted automatically

### Exit Criteria
- PR webhook triggers automatic review comment
- Deploy status queryable via API
- All tests pass

---

## Phase 5 — Hardening & Production Readiness

**Duration:** 2 weeks  
**Goal:** System is production-ready, observable, and documented

### Deliverables

- [ ] **Observability:** Structured logging across all components (correlation_id tracing)
- [ ] **Health checks:** `/health` on all services, aggregated health dashboard
- [ ] **Rate limiting:** Configurable per-caller rate limits in gateway
- [ ] **Alert deduplication:** Redis-based dedup for P1 webhook storms
- [ ] **Retry logic:** Exponential backoff on all MCP calls
- [ ] **Circuit breaker:** Skip non-critical skills if MCP server is down
- [ ] **Load testing:** Simulate 50 concurrent P1 alerts
- [ ] **Security review:** Review with org security team
- [ ] **Runbooks:** Operational runbook for on-call team
- [ ] **Deployment manifests:** Kubernetes/Docker Compose configs for all services

### Exit Criteria
- System handles 50 concurrent alerts without degradation
- Passes org security review
- Operations team signed off on runbook
- All components have health checks registered in monitoring

---

## Phase Gate: Nexus Validation

Before each phase starts, the lead developer must confirm all new dependencies for that phase are available in the org Nexus repository. See [DEPENDENCIES.md](DEPENDENCIES.md) for the full list organized by phase.

If a dependency is unavailable in Nexus, options are:
1. Request Nexus team to add it (preferred — submit request 2 weeks ahead)
2. Identify an alternative package with equivalent functionality
3. Implement the functionality from scratch using only available packages
