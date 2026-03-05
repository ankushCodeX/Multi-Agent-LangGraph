# Organizational Constraints & Design Decisions

This document records every organizational constraint we are designing around, and the explicit architectural decision made in response to each one. Anyone joining the project should read this first.

---

## Constraint 1 — Package Availability (Nexus Only)

**Constraint:**  
The organization's development environment only allows packages from the internal Nexus repository. PyPI and npm are not directly accessible. Any package not in Nexus cannot be used.

**Design Decision:**  
- A mandatory `DEPENDENCIES.md` is maintained listing every package with its version
- No new package may be added to the codebase without first confirming it is in Nexus
- The Nexus confirmation checkbox in `DEPENDENCIES.md` must be ticked before a package is imported anywhere in code
- If a required package is missing from Nexus, the team must either: (a) request Nexus team to add it with 2 weeks lead time, (b) find an alternative already in Nexus, or (c) implement the needed function from scratch using stdlib only

**Package selection philosophy:**  
We have deliberately chosen packages with minimal transitive dependencies. `langgraph` was chosen over the full `langchain` stack specifically to reduce the Nexus approval surface area.

---

## Constraint 2 — Model Agnostic (Org Model Directory)

**Constraint:**  
Claude (Anthropic) may not be available or approved in the organization's Model Directory. The LLM available today may change in future. No assumption can be made about which LLM is accessible.

**Design Decision:**  
- An `LLMAdapter` abstract base class is the sole interface to any LLM
- No agent, skill, orchestrator, or MCP server ever calls an LLM API directly
- The concrete adapter implementation (e.g. `AzureOpenAIAdapter`, `BedrockAdapter`) is configured entirely via environment variables
- Switching LLM providers requires **zero code changes** — only environment variable changes
- The `LLM_PROVIDER` environment variable selects the active adapter at startup
- All concrete adapters are maintained in `src/llm-adapter/` — one file per provider
- See `COMPONENT_CONTRACTS.md` for the full `LLMAdapter` interface

**Adapters to build (in priority order based on org approval likelihood):**
1. `AzureOpenAIAdapter` — most likely to be org-approved
2. `BedrockAdapter` — if org is AWS-based
3. `OllamaAdapter` — for local/on-prem model deployment
4. `AnthropicAdapter` — if Claude gets approved
5. `OpenAICompatibleAdapter` — generic fallback for any OpenAI-spec endpoint

---

## Constraint 3 — Each Tool Requires Its Own MCP Server

**Constraint:**  
There is no shared or community MCP server available inside the organization. Each internal tool (Splunk, AppDynamics, Confluence, etc.) requires a custom MCP server to be built and deployed.

**Design Decision:**  
- Each MCP server lives in its own directory: `src/mcp-servers/{tool-name}/`
- Each MCP server is a completely independent deployable unit (its own `package.json`, its own Docker image, its own health check)
- No two MCP servers share code — shared utility functions are copied, not imported cross-server (to prevent deployment coupling)
- Each MCP server exposes a `/describe` endpoint so the registry can validate it at startup
- The MCP server registry (`registry.yaml`) is the only place that knows about all MCP servers
- MCP servers are built in TypeScript using `@modelcontextprotocol/sdk` — this is the reference implementation and most stable

**MCP Server build checklist (each server must have):**
- [ ] `src/index.ts` — server entry point
- [ ] `src/tools/` — one file per tool
- [ ] `src/client.ts` — HTTP client for the underlying tool API
- [ ] `tests/` — unit tests with mocked HTTP responses
- [ ] `Dockerfile` — standalone container
- [ ] `README.md` — tool-specific setup and auth instructions
- [ ] `/health` endpoint
- [ ] `/describe` endpoint

---

## Constraint 4 — Hybrid Deployment (On-prem + Cloud)

**Constraint:**  
Sensitive internal tools (Splunk, AppDynamics, Confluence) run on-prem. SaaS tools (PagerDuty, Slack, Jira Cloud) run in cloud. The system must operate across both zones without compromising data boundaries.

**Design Decision:**  
- The FastAPI gateway is the sole boundary between zones
- The orchestrator, all agents, all skills, and all on-prem MCP servers run on-prem
- Outbound HTTPS calls exit only from: the LLM Adapter (to org Model Directory) and cloud MCP servers (to SaaS tools)
- Raw internal data (logs, metrics, internal docs) never leaves on-prem
- LLM prompts contain only sanitized summaries — a `sanitize_for_llm()` utility is provided
- All secrets are injected via environment variables from org vault — never in config files or code
- See `HYBRID_DEPLOYMENT.md` for full zone diagram and network requirements

---

## Constraint 5 — Future Org Agent Provider Platform

**Constraint:**  
The organization may deploy a common Agent Provider platform in future. When that happens, this project's MCP servers and agents should be publishable to that platform without rewriting.

**Design Decision:**  
- MCP servers are built as standalone services with standard `/describe` and `/health` endpoints — they are already publishable to any MCP-compatible platform
- Agents are defined as configuration (system prompt + allowed skills list) — not hardcoded logic. They can be ported to any agent platform
- The orchestrator is the only component that is "opinionated" (LangGraph). It is isolated behind the `Agent Router` interface — if replaced by an org platform router, everything below it (skills, MCP servers) remains unchanged
- Skills are callable via a typed Python interface AND will have an optional REST wrapper added in Phase 5 — making them callable from any platform
- The `AGENT_PROVIDER_GUIDE.md` documents the exact steps to hand off to the org platform when ready

**Transition strategy when org platform arrives:**
1. Register MCP servers directly (they already have `/describe` endpoints)
2. Port agent definitions (system prompt + skill list) to org platform's format
3. Redirect orchestrator's agent routing to call org platform instead of local agents
4. Retire local agent implementations
5. Keep skills library and MCP servers unchanged

---

## Constraint 6 — Independent Testability

**Constraint:**  
Each component must be testable in complete isolation — no live external dependencies, no running Redis, no live LLM, no real tool APIs required.

**Design Decision:**  
- A `MockLLMAdapter` is provided in `tests/mocks/` — returns fixture responses
- A `MockMCPClient` is provided in `tests/mocks/` — returns fixture MCP results
- A `MockMCPServer` (lightweight FastAPI app) is provided in `tests/mocks/` — simulates any MCP server
- Redis is wrapped in a `StateStore` interface with a `MockStateStore` for tests
- All external HTTP calls go through `httpx` — `respx` library enables request mocking in tests
- No test in `tests/unit/` may make a real network call — enforced by pytest plugin
- `tests/integration/` tests use mock MCP servers and mock LLM only — still no real external services
- Only `tests/e2e/` tests are allowed to call real services — and only in a designated test environment
- See `TESTING_GUIDE.md` for the complete testing strategy and patterns

**Test isolation rule enforcement:**
```python
# conftest.py — applied to all unit tests automatically
@pytest.fixture(autouse=True)
def block_real_http(respx_mock):
    """Fails any test that makes a real HTTP call."""
    yield
```

---

## Decision Log

| Date | Decision | Rationale | Alternatives Considered |
|---|---|---|---|
| 2026-03 | LangGraph over CrewAI | More control over execution graph; better for complex P1 alert flows | CrewAI (less control), AutoGen (heavy, Microsoft-specific) |
| 2026-03 | TypeScript for MCP servers | MCP SDK reference implementation is TypeScript; most stable | Python MCP SDK (less mature at time of decision) |
| 2026-03 | FastAPI for gateway | Lightweight, async, excellent Pydantic integration | Django (too heavy), Flask (no async) |
| 2026-03 | Redis for state | Hybrid deployment compatible; fast; widely available in orgs | Postgres (heavier), in-memory (not persistent across restarts) |
| 2026-03 | Pydantic v2 for contracts | Best-in-class Python data validation; native JSON schema export | dataclasses (no validation), attrs (less ecosystem) |
| 2026-03 | One MCP server per tool | Org constraint; also better isolation and independent deployability | Shared MCP server (coupling risk) |
