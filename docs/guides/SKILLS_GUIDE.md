# Skills Library Guide

## What is a Skill?

A skill is the atomic unit of capability in this system. It is a Python class that:
- Accepts a typed, validated input
- Calls one or more MCP servers via `MCPClient`
- Optionally calls the LLM Adapter for reasoning
- Returns a typed, validated output

Skills are what agents invoke. Agents never call MCP servers or the LLM directly.

```
Agent
  └── invokes Skill
        ├── calls MCPClient → MCP Server → Tool API
        └── calls LLMAdapter → LLM API (optional)
```

---

## Skill Directory Structure

```
src/skills/
├── base.py                        ← Abstract Skill base class (do not modify)
├── observability/
│   ├── __init__.py
│   ├── alert_triage.py            ← Skill implementation
│   ├── alert_triage_schema.py     ← Input/output Pydantic models
│   └── tests/
│       └── test_alert_triage.py   ← Unit tests (mock MCP + mock LLM)
├── workflow/
├── knowledge/
└── sdlc/
```

---

## Step-by-Step: Writing a New Skill

### Step 1 — Define input and output models

Create `{skill_name}_schema.py`:

```python
from pydantic import BaseModel, Field
from src.shared.contracts import SkillInput, SkillOutput

class AlertTriageInput(SkillInput):
    alert_type: str = Field(description="Type of alert e.g. CPU_HIGH, OOM, LATENCY")
    service_name: str = Field(description="Name of the affected service")
    severity: str = Field(description="critical | warning | info")
    alert_payload: dict = Field(description="Raw alert payload from the monitoring tool")
    time_range_minutes: int = Field(default=30, description="Log lookback window in minutes")

class AlertTriageOutput(SkillOutput):
    summary: str = Field(description="Human-readable summary of the alert")
    likely_cause: str = Field(description="LLM-assessed likely root cause")
    affected_services: list[str] = Field(description="Services impacted")
    log_evidence: list[str] = Field(description="Relevant log lines found")
    recommended_action: str = Field(description="Suggested next step")
    confidence: float = Field(ge=0.0, le=1.0, description="LLM confidence 0-1")
```

### Step 2 — Implement the skill

```python
# alert_triage.py
from src.shared.base_skill import Skill
from src.shared.mcp_client import MCPClient
from src.shared.llm_adapter import LLMAdapter, LLMMessage, LLMRequest
from .alert_triage_schema import AlertTriageInput, AlertTriageOutput

class AlertTriageSkill(Skill):

    def __init__(self, mcp_client: MCPClient, llm_adapter: LLMAdapter):
        # Dependencies injected — never instantiated inside the skill
        self._mcp = mcp_client
        self._llm = llm_adapter

    @property
    def name(self) -> str:
        return "alert_triage"

    @property
    def description(self) -> str:
        return "Triages an incoming alert by correlating logs and metrics, then provides a root cause summary"

    def execute(self, input: AlertTriageInput) -> AlertTriageOutput:

        # Step 1: Fetch alert details from AppDynamics
        alert_details = self._mcp.call_tool(
            server_name="appdynamics-mcp",
            tool_name="get_alert_details",
            arguments={"alert_id": input.alert_payload.get("alert_id")},
        )

        # Step 2: Search related logs in Splunk
        log_results = self._mcp.call_tool(
            server_name="splunk-mcp",
            tool_name="search_logs",
            arguments={
                "query": f"service={input.service_name} severity=error",
                "service_name": input.service_name,
                "time_range_minutes": input.time_range_minutes,
            },
        )

        # Step 3: Use LLM to reason over the findings
        llm_response = self._llm.complete(LLMRequest(
            messages=[
                LLMMessage(role="system", content=(
                    "You are a DevOps incident analyst. "
                    "Analyze the alert and log data provided and return a JSON object with: "
                    "summary, likely_cause, affected_services (array), recommended_action, confidence (0-1)."
                )),
                LLMMessage(role="user", content=(
                    f"Alert type: {input.alert_type}\n"
                    f"Service: {input.service_name}\n"
                    f"Severity: {input.severity}\n"
                    f"Alert details: {alert_details.data}\n"
                    f"Recent error logs: {log_results.data.get('events', [])[:10]}\n"
                    "Analyze and return JSON."
                )),
            ],
            max_tokens=800,
            temperature=0.1,
        ))

        import json
        analysis = json.loads(llm_response.content)

        return AlertTriageOutput(
            skill_name=self.name,
            status="success",
            data=analysis,
            mcp_calls_made=2,
            summary=analysis["summary"],
            likely_cause=analysis["likely_cause"],
            affected_services=analysis.get("affected_services", [input.service_name]),
            log_evidence=[e["message"] for e in log_results.data.get("events", [])[:5]],
            recommended_action=analysis["recommended_action"],
            confidence=analysis["confidence"],
        )
```

### Step 3 — Write tests

All tests use mocked MCP and mocked LLM — no real network calls.

```python
# tests/test_alert_triage.py
import pytest
from unittest.mock import MagicMock
from src.skills.observability.alert_triage import AlertTriageSkill
from src.skills.observability.alert_triage_schema import AlertTriageInput
from src.shared.mcp_client import MCPResult
from src.shared.llm_adapter import LLMResponse

@pytest.fixture
def mock_mcp():
    mcp = MagicMock()
    # AppDynamics response
    mcp.call_tool.side_effect = [
        MCPResult(server="appdynamics-mcp", tool="get_alert_details", status="success",
                  data={"alert_id": "123", "tier": "payment-service"}, duration_ms=50),
        # Splunk response
        MCPResult(server="splunk-mcp", tool="search_logs", status="success",
                  data={"events": [{"message": "OOMKilled", "timestamp": "2026-01-01", "severity": "error"}]},
                  duration_ms=120),
    ]
    return mcp

@pytest.fixture
def mock_llm():
    llm = MagicMock()
    llm.complete.return_value = LLMResponse(
        content='{"summary": "Payment service OOM", "likely_cause": "Memory leak", '
                '"affected_services": ["payment-service"], '
                '"recommended_action": "Restart pod, check heap dump", "confidence": 0.85}',
        input_tokens=200,
        output_tokens=80,
    )
    return llm

def test_alert_triage_success(mock_mcp, mock_llm):
    skill = AlertTriageSkill(mcp_client=mock_mcp, llm_adapter=mock_llm)
    result = skill.execute(AlertTriageInput(
        task_id="task-001",
        alert_type="OOM",
        service_name="payment-service",
        severity="critical",
        alert_payload={"alert_id": "123"},
    ))

    assert result.status == "success"
    assert result.likely_cause == "Memory leak"
    assert result.confidence == 0.85
    assert mock_mcp.call_tool.call_count == 2
    assert mock_llm.complete.call_count == 1

def test_alert_triage_mcp_failure_is_handled(mock_mcp, mock_llm):
    mock_mcp.call_tool.side_effect = [
        MCPResult(server="appdynamics-mcp", tool="get_alert_details",
                  status="error", data=None, error_message="timeout", duration_ms=5000),
        MCPResult(server="splunk-mcp", tool="search_logs", status="success",
                  data={"events": []}, duration_ms=100),
    ]
    skill = AlertTriageSkill(mcp_client=mock_mcp, llm_adapter=mock_llm)
    # Should not raise — should degrade gracefully
    result = skill.execute(AlertTriageInput(
        task_id="task-002",
        alert_type="CPU_HIGH",
        service_name="auth-service",
        severity="warning",
        alert_payload={},
    ))
    assert result.status in ("success", "partial")
```

---

## Skill Registration

After writing a skill, register it in the Skills Registry so agents can discover it:

```python
# src/skills/registry.py
from src.skills.observability.alert_triage import AlertTriageSkill
from src.skills.observability.log_analysis import LogAnalysisSkill
# ... import others

SKILL_REGISTRY: dict[str, type] = {
    "alert_triage": AlertTriageSkill,
    "log_analysis": LogAnalysisSkill,
    # add new skills here
}

def get_skill(name: str, mcp_client, llm_adapter) -> Skill:
    if name not in SKILL_REGISTRY:
        raise ValueError(f"Unknown skill: {name}. Available: {list(SKILL_REGISTRY.keys())}")
    return SKILL_REGISTRY[name](mcp_client=mcp_client, llm_adapter=llm_adapter)
```

---

## Skills by Domain

### Observability (P1)

| Skill Name | MCP Servers Used | LLM Used? | Description |
|---|---|---|---|
| `alert_triage` | appdynamics-mcp, splunk-mcp | Yes | Correlate alert with logs, produce root cause |
| `log_analysis` | splunk-mcp | Yes | Deep log analysis for a service and time range |
| `metric_correlation` | appdynamics-mcp | Yes | Correlate metrics across services |
| `incident_creator` | pagerduty-mcp, jira-mcp | No | Create incident from structured triage result |
| `runbook_lookup` | confluence-mcp | No | Find runbook matching alert type |

### Workflow (P2)

| Skill Name | MCP Servers Used | LLM Used? | Description |
|---|---|---|---|
| `task_router` | jira-mcp | Yes | Route task to correct team/queue |
| `approval_chain` | jira-mcp, slack-mcp | No | Trigger approval flow |
| `escalation_handler` | jira-mcp, slack-mcp | No | Escalate stale items |
| `status_broadcaster` | slack-mcp | No | Post status update to channels |

### Knowledge (P3)

| Skill Name | MCP Servers Used | LLM Used? | Description |
|---|---|---|---|
| `confluence_search` | confluence-mcp | No | Search across Confluence spaces |
| `doc_summarizer` | confluence-mcp | Yes | Fetch and summarize a document |
| `runbook_generator` | confluence-mcp, jira-mcp | Yes | Generate runbook from incident history |

### SDLC (P4)

| Skill Name | MCP Servers Used | LLM Used? | Description |
|---|---|---|---|
| `pr_reviewer` | github-mcp | Yes | Review PR diff, post structured comment |
| `code_context` | github-mcp | Yes | Explain code change for non-developers |
| `deploy_checker` | github-mcp | No | Check pipeline / deployment status |

---

## Skill Checklist

Before marking a skill complete:

- [ ] Input model extends `SkillInput` with all fields documented
- [ ] Output model extends `SkillOutput` with all fields documented
- [ ] Skill class constructor accepts `mcp_client` and `llm_adapter` as injected dependencies
- [ ] No direct HTTP calls, no direct LLM imports — only via injected adapters
- [ ] Graceful handling when MCP returns `status=error` or `status=timeout`
- [ ] Unit tests cover: success, MCP failure, LLM failure, invalid input
- [ ] No test makes a real network call
- [ ] Skill registered in `src/skills/registry.py`
