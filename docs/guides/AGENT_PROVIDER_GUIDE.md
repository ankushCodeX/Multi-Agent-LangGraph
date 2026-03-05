# Agent Provider Integration Guide

## Purpose

This guide explains how the current system is structured to support a future handoff to an organizational Agent Provider platform — and what steps to take when that platform becomes available.

The architecture has been designed from day one so that **no component needs to be rewritten** when transitioning. MCP servers are already publishable. Agents are portable by design. Only the orchestrator's routing layer needs to be redirected.

---

## Current Architecture vs Future Architecture

### Current (Self-Hosted)
```
User/Trigger
    └── API Gateway
          └── Orchestrator (LangGraph)
                └── Agent Router → Specialist Agents
                      └── Skills → MCP Servers → Tools
```

### Future (Org Agent Provider Platform)
```
User/Trigger
    └── API Gateway (optional — org platform may provide this)
          └── Org Agent Provider Platform
                ├── Registered MCP Servers ← (your servers, unchanged)
                └── Registered Agent Definitions ← (your agents, re-registered)
                      └── Skills → MCP Servers → Tools (unchanged)
```

The Skills Library and MCP Servers are **completely unchanged**. Only the top of the stack is handed off.

---

## What Each Component Looks Like to the Org Platform

### MCP Servers — Already Platform-Ready

Your MCP servers already implement the contract the org platform needs:

| Endpoint | Purpose | Status |
|---|---|---|
| `/health` | Platform health checks | ✅ Built in Phase 1 |
| `/describe` | Auto-discovery of tools | ✅ Built in Phase 1 |
| MCP protocol on stdio/HTTP | Tool invocation | ✅ Core MCP SDK |

**To register with the org platform:**
1. Provide the MCP server URL (must be reachable from the platform)
2. Platform calls `/describe` to discover tools automatically
3. Done — no code changes

### Agent Definitions — Portable by Design

Each agent in this system is defined as configuration, not hardcoded logic:

```python
# src/agents/devops_agent.py — what it actually is
DEVOPS_AGENT_CONFIG = {
    "name": "DevOps Agent",
    "role": "Specialist in observability, incident triage, and DevOps automation",
    "system_prompt": """
        You are a DevOps incident analyst. Your role is to:
        1. Analyze incoming alerts and correlate with log data
        2. Identify root causes using available observability tools
        3. Create incidents and notify the appropriate teams
        4. Look up and follow runbooks for known alert patterns
        Always prefer data-backed conclusions over speculation.
    """,
    "allowed_skills": [
        "alert_triage",
        "log_analysis",
        "metric_correlation",
        "incident_creator",
        "runbook_lookup",
    ],
}
```

To port to an org platform: copy `system_prompt` and `allowed_skills` into the platform's agent definition format. The logic is in the skills — not in the agent.

### Skills — Callable via REST (Phase 5)

In Phase 5, each skill will get a lightweight REST wrapper, making it callable from any platform:

```
POST /skills/alert_triage
Content-Type: application/json

{
  "task_id": "...",
  "alert_type": "OOM",
  "service_name": "payment-service",
  ...
}
```

This means the org platform can invoke skills directly as tools — without going through the local orchestrator.

---

## Transition Checklist

When the org Agent Provider platform is ready:

### Phase T1 — Register MCP Servers
- [ ] Confirm org platform supports MCP protocol version in use
- [ ] Ensure all MCP server `/describe` endpoints return valid JSON
- [ ] Register each MCP server URL in the org platform's registry
- [ ] Verify the platform can reach on-prem MCP servers (network routing)
- [ ] Smoke test each registered tool from the platform

### Phase T2 — Register Agent Definitions
- [ ] Map `DEVOPS_AGENT_CONFIG` → org platform agent format
- [ ] Map `WORKFLOW_AGENT_CONFIG` → org platform agent format
- [ ] Map `KNOWLEDGE_AGENT_CONFIG` → org platform agent format
- [ ] Map `SDLC_AGENT_CONFIG` → org platform agent format
- [ ] Map allowed skills to corresponding MCP server tools in platform

### Phase T3 — Redirect Orchestrator Routing
- [ ] Add `OrgPlatformAgentRouter` implementing the `AgentRouter` interface
- [ ] Configure with org platform API endpoint and credentials
- [ ] Toggle via environment variable: `AGENT_ROUTER=org_platform`
- [ ] Run full integration test suite against the org platform routing
- [ ] Run in parallel (shadow mode) for 1 week before cutting over

### Phase T4 — Decommission Local Components
- [ ] Disable local `LangGraphOrchestrator` (keep code for rollback)
- [ ] Retire local agent implementations
- [ ] Keep Skills Library and MCP Servers unchanged — they are now platform tools
- [ ] Update `ARCHITECTURE.md` to reflect new topology

---

## Interface: AgentRouter (the seam)

The `AgentRouter` abstraction is the single interface that separates the orchestrator from the agent execution layer. Swapping to an org platform means implementing this one interface:

```python
# src/orchestrator/agent_router.py
from abc import ABC, abstractmethod

class AgentRouter(ABC):
    @abstractmethod
    def route(self, agent_input: AgentInput) -> AgentOutput:
        """Route a sub-task to the appropriate agent and return its output."""
        ...

# Current implementation
class LocalAgentRouter(AgentRouter):
    def route(self, agent_input: AgentInput) -> AgentOutput:
        agent = self._get_agent(agent_input.intent)
        return agent.handle(agent_input)

# Future implementation — only this changes
class OrgPlatformAgentRouter(AgentRouter):
    def __init__(self, platform_url: str, api_key: str):
        self._platform_url = platform_url
        self._api_key = api_key

    def route(self, agent_input: AgentInput) -> AgentOutput:
        # Call org platform API with agent_input
        # Receive response, map to AgentOutput
        ...
```

**Environment variable toggle:**
```bash
AGENT_ROUTER=local           # uses LocalAgentRouter (default)
AGENT_ROUTER=org_platform    # uses OrgPlatformAgentRouter
ORG_PLATFORM_URL=https://...
ORG_PLATFORM_API_KEY=...
```

---

## Questions to Ask the Org Platform Team

When the platform becomes available, gather these details before starting the transition:

1. Which version of the MCP protocol does the platform support?
2. Does the platform support on-prem MCP servers, or only cloud-hosted?
3. What is the agent definition format (JSON schema, YAML, UI-based)?
4. How are skills/tools authorized per agent?
5. Does the platform provide its own API Gateway, or do we keep ours?
6. What observability/logging does the platform provide?
7. Is there a staging environment to test before production?
8. How are secrets (MCP tokens) managed — does the platform provide a vault?
