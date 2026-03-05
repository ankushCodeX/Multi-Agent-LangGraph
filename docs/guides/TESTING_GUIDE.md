# Testing Guide

## Philosophy

Every component in this system must be testable in **complete isolation** — no live Redis, no live LLM, no real tool APIs. This is non-negotiable given org network constraints and the need for CI to run on locked-down machines.

---

## Test Layers

```
tests/
├── unit/           ← Fastest. One component. All dependencies mocked.
├── integration/    ← Medium. Multiple components. Mock external services only.
└── e2e/            ← Slowest. Full flow. Runs only in dedicated test environment.
```

**Rule:** `pytest tests/unit/` and `pytest tests/integration/` must pass on any developer machine with no network access.

---

## Available Mocks (use these, don't write your own)

All shared mocks live in `tests/mocks/`.

### MockLLMAdapter
Returns a fixture response. Configurable per test.

```python
from tests.mocks.mock_llm_adapter import MockLLMAdapter

# Default — returns empty response
llm = MockLLMAdapter()

# Configure a specific response
llm = MockLLMAdapter(response_content='{"summary": "test", "confidence": 0.9}')

# Configure to raise an error
llm = MockLLMAdapter(raise_error=True)

# Inspect what was called
assert llm.call_count == 1
assert "payment-service" in llm.last_request.messages[-1].content
```

### MockMCPClient
Returns fixture results per (server, tool) pair.

```python
from tests.mocks.mock_mcp_client import MockMCPClient
from src.shared.mcp_client import MCPResult

mcp = MockMCPClient()

# Register a response for a specific (server, tool) call
mcp.register_response(
    server_name="splunk-mcp",
    tool_name="search_logs",
    result=MCPResult(
        server="splunk-mcp",
        tool="search_logs",
        status="success",
        data={"events": [{"message": "OOMKilled", "timestamp": "2026-01-01"}]},
        duration_ms=100,
    )
)

# Register an error response
mcp.register_error("appdynamics-mcp", "get_alert_details", "Connection timeout")

# Inspect calls made
assert mcp.call_count == 2
assert mcp.was_called("splunk-mcp", "search_logs")
```

### MockStateStore
In-memory Redis replacement.

```python
from tests.mocks.mock_state_store import MockStateStore

state = MockStateStore()
state.set("task:123", {"status": "running"})
assert state.get("task:123") == {"status": "running"}
```

### MockMCPServer (for integration tests)
A real lightweight FastAPI server that simulates an MCP server. Used in integration tests where you want the full HTTP stack.

```python
from tests.mocks.mock_mcp_server import MockMCPServer

server = MockMCPServer(name="splunk-mcp")
server.register_tool_response("search_logs", {"events": [], "total": 0})

with server.running(port=9001):
    # MCP client will connect to localhost:9001
    result = mcp_client.call_tool("splunk-mcp", "search_logs", {...})
    assert result.status == "success"
```

---

## Testing Each Component

### Testing a Skill

```python
# Pattern: inject mocks, call execute(), assert output
def test_my_skill():
    mcp = MockMCPClient()
    mcp.register_response("my-mcp", "my_tool", MCPResult(...))

    llm = MockLLMAdapter(response_content='{"result": "ok"}')

    skill = MySkill(mcp_client=mcp, llm_adapter=llm)
    output = skill.execute(MySkillInput(task_id="t1", ...))

    assert output.status == "success"
    assert mcp.was_called("my-mcp", "my_tool")
```

**What to test for every skill:**
- Success path — all MCP calls succeed, LLM responds correctly
- MCP error — one MCP call returns `status=error`; skill should degrade gracefully
- MCP timeout — one MCP call returns `status=timeout`
- LLM error — LLM adapter raises exception; skill should handle it
- Invalid input — Pydantic validation catches bad input before execute() is called

---

### Testing an Agent

```python
# Pattern: inject mock skill registry, call agent, assert skills invoked
def test_devops_agent_calls_alert_triage():
    mock_skill_registry = MockSkillRegistry()
    mock_skill_registry.register("alert_triage", returns=AlertTriageOutput(
        skill_name="alert_triage", status="success", data={}, mcp_calls_made=2,
        summary="OOM in payment-service", likely_cause="Memory leak",
        affected_services=["payment-service"], log_evidence=[],
        recommended_action="Restart pod", confidence=0.9,
    ))

    agent = DevOpsAgent(skill_registry=mock_skill_registry)
    output = agent.handle(AgentInput(
        task_id="t1", sub_task_id="s1",
        intent="observability",
        instruction="Triage this critical alert",
        context={"alert_type": "OOM", "service_name": "payment-service", "severity": "critical"},
        allowed_skills=["alert_triage", "incident_creator"],
        session_id=None,
    ))

    assert output.status == "completed"
    assert "alert_triage" in output.skills_used
```

---

### Testing the Orchestrator

```python
# Pattern: mock agent router outputs, verify orchestrator assembles TaskResponse
def test_orchestrator_routes_p1_alert():
    mock_router = MockAgentRouter()
    mock_router.set_response(AgentOutput(
        sub_task_id="s1", status="completed",
        result={"summary": "Incident created: PD-1234"},
        skills_used=["alert_triage", "incident_creator"],
        llm_calls_made=1,
    ))

    orchestrator = Orchestrator(agent_router=mock_router, state_store=MockStateStore())
    response = orchestrator.process(TaskRequest(
        task_id="t1", source="webhook", priority="P1",
        context={"normalized_text": "Critical OOM alert on payment-service",
                 "raw_payload": {}},
        timestamp="2026-01-01T00:00:00Z",
    ))

    assert response.status == "completed"
    assert response.result.summary != ""
```

---

### Testing an MCP Server (TypeScript)

MCP servers are tested with `jest` + `nock` to mock HTTP calls to the underlying tool API.

```typescript
// tests/tools/search_logs.test.ts
import nock from "nock";
import { ToolClient } from "../../src/client";
import { searchLogs } from "../../src/tools/search_logs";

describe("searchLogs tool", () => {
  const client = new ToolClient({ baseUrl: "https://splunk-internal", token: "tok" });

  beforeEach(() => nock.cleanAll());

  it("success: returns formatted events", async () => {
    nock("https://splunk-internal")
      .get("/services/search/jobs/export")
      .query(true)
      .reply(200, {
        count: 1,
        events: [{ _time: "2026-01-01", _raw: "OOMKilled", severity: "error" }],
        search_url: "https://splunk/s/1",
      });

    const result = await searchLogs({ query: "error", service_name: "svc", time_range_minutes: 5 }, client);
    expect(result.total_events).toBe(1);
    expect(result.events[0].message).toBe("OOMKilled");
  });

  it("error: throws on 401 unauthorized", async () => {
    nock("https://splunk-internal").get("/services/search/jobs/export").query(true).reply(401);
    await expect(
      searchLogs({ query: "test", service_name: "svc", time_range_minutes: 5 }, client)
    ).rejects.toThrow(/unauthorized/i);
  });

  it("error: throws on network timeout", async () => {
    nock("https://splunk-internal").get("/services/search/jobs/export")
      .query(true).replyWithError({ code: "ETIMEDOUT" });
    await expect(
      searchLogs({ query: "test", service_name: "svc", time_range_minutes: 5 }, client)
    ).rejects.toThrow();
  });
});
```

---

## Running Tests

### Unit tests (no network, fast)
```bash
# Python — all unit tests
pytest tests/unit/ -v

# Python — single component
pytest tests/unit/skills/test_alert_triage.py -v

# TypeScript — single MCP server
cd src/mcp-servers/splunk && npm test

# TypeScript — all MCP servers
for dir in src/mcp-servers/*/; do (cd "$dir" && npm test); done
```

### Integration tests
```bash
# Starts mock MCP servers locally, then runs integration suite
pytest tests/integration/ -v
```

### Coverage
```bash
pytest tests/unit/ --cov=src --cov-report=term-missing
```

### Enforcing no real HTTP in unit tests

`tests/conftest.py` (applied automatically to all unit tests):
```python
import pytest
import respx

@pytest.fixture(autouse=True)
def no_real_http(respx_mock):
    """Any real HTTP call in a unit test will raise an error."""
    yield
```

---

## CI Pipeline (No External Network)

The CI pipeline must run entirely without external network access:

```yaml
# .github/workflows/ci.yml (or equivalent internal CI config)
steps:
  - name: Install Python deps from Nexus
    run: pip install -r requirements.txt --index-url ${NEXUS_PYPI_URL}

  - name: Install Node deps from Nexus
    run: npm ci --registry=${NEXUS_NPM_URL}
    working-directory: src/mcp-servers/splunk

  - name: Lint (Python)
    run: ruff check src/

  - name: Type check (Python)
    run: mypy src/

  - name: Unit tests (Python)
    run: pytest tests/unit/ --cov=src --cov-fail-under=80

  - name: Unit tests (TypeScript — per MCP server)
    run: |
      for dir in src/mcp-servers/*/; do
        (cd "$dir" && npm test -- --coverage)
      done
```

**Coverage targets:**
- Skills: 90% minimum
- Agents: 85% minimum
- Orchestrator: 85% minimum
- MCP server tools: 90% minimum
- Shared contracts/adapters: 95% minimum
