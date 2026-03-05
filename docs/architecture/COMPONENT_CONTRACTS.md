# Component Contracts

Every component in this system communicates through defined contracts. No component ever depends on the internal implementation of another. This document is the authoritative reference for all inter-component interfaces.

---

## Contract Index

| Contract | Between | File |
|---|---|---|
| TaskRequest | Trigger Layer → Gateway → Orchestrator | [#taskrequest](#taskrequest) |
| TaskResponse | Orchestrator → Gateway → Caller | [#taskresponse](#taskresponse) |
| AgentInput / AgentOutput | Orchestrator → Agent | [#agentio](#agent-inputoutput) |
| SkillInput / SkillOutput | Agent → Skill | [#skillio](#skill-inputoutput) |
| MCPCall / MCPResult | Skill → MCP Server | [#mcpio](#mcp-call--result) |
| LLMRequest / LLMResponse | Any → LLM Adapter | [#llmio](#llm-adapter-contract) |

---

## TaskRequest

Produced by the Trigger Layer. Consumed by the Gateway and Orchestrator.

```json
{
  "task_id": "string (UUID v4)",
  "source": "webhook | slack | api | cli",
  "priority": "P1 | P2 | P3 | P4",
  "context": {
    "raw_payload": "object (source-specific)",
    "normalized_text": "string (human-readable description of the task)",
    "metadata": {
      "service": "string (optional)",
      "environment": "string (optional)",
      "triggered_by": "string (optional)"
    }
  },
  "timestamp": "ISO8601 string",
  "correlation_id": "string (optional, for tracing)"
}
```

**Validation rules:**
- `task_id` must be unique — duplicates within 5 minutes are deduplicated via Redis
- `priority` defaults to P3 if not provided by trigger
- `context.normalized_text` is required — triggers must normalize before sending

---

## TaskResponse

Produced by the Orchestrator. Returned to the caller via the Gateway.

```json
{
  "task_id": "string (matches request)",
  "status": "completed | partial | failed",
  "result": {
    "summary": "string (human-readable outcome)",
    "actions_taken": [
      {
        "skill": "string",
        "outcome": "string",
        "reference": "string (e.g. JIRA-1234, PD-5678)"
      }
    ],
    "errors": [
      {
        "component": "string",
        "message": "string",
        "recoverable": "boolean"
      }
    ]
  },
  "duration_ms": "integer",
  "timestamp": "ISO8601 string"
}
```

---

## Agent Input/Output

The contract between the Orchestrator's Agent Router and each Specialist Agent.

**AgentInput:**
```json
{
  "task_id": "string",
  "sub_task_id": "string",
  "intent": "observability | workflow | knowledge | sdlc",
  "instruction": "string (natural language task description)",
  "context": "object (task-specific structured context)",
  "allowed_skills": ["string array — skill names this agent may call"],
  "session_id": "string (for stateful handoffs via Redis)"
}
```

**AgentOutput:**
```json
{
  "sub_task_id": "string",
  "status": "completed | failed | needs_escalation",
  "result": "object (skill-defined structure)",
  "skills_used": ["string array"],
  "llm_calls_made": "integer",
  "handoff_to": "string (optional — agent name if escalating)"
}
```

**Rules:**
- An agent MUST only invoke skills listed in `allowed_skills`
- An agent MUST NOT call LLM directly — only via LLM Adapter
- An agent MUST NOT call MCP servers directly — only via Skills

---

## Skill Input/Output

The contract every skill must implement. All skills extend the base `Skill` abstract class.

**Base Skill Interface (Python):**
```python
from abc import ABC, abstractmethod
from pydantic import BaseModel

class SkillInput(BaseModel):
    task_id: str
    session_id: str | None = None
    # skill-specific fields defined in subclass

class SkillOutput(BaseModel):
    skill_name: str
    status: str  # "success" | "partial" | "failed"
    data: dict
    error: str | None = None
    mcp_calls_made: int = 0

class Skill(ABC):
    @property
    @abstractmethod
    def name(self) -> str: ...

    @property
    @abstractmethod
    def description(self) -> str: ...

    @abstractmethod
    def execute(self, input: SkillInput) -> SkillOutput: ...
```

**Example — AlertTriageInput:**
```python
class AlertTriageInput(SkillInput):
    alert_type: str
    service_name: str
    severity: str  # "critical" | "warning" | "info"
    alert_payload: dict
    time_range_minutes: int = 30
```

**Example — AlertTriageOutput:**
```python
class AlertTriageOutput(SkillOutput):
    summary: str
    likely_cause: str
    affected_services: list[str]
    log_evidence: list[str]
    recommended_action: str
    confidence: float  # 0.0 - 1.0
```

---

## MCP Call / Result

The contract between Skills and MCP Servers. All MCP communication goes through the `MCPClient` wrapper — skills never instantiate raw MCP connections.

**MCPClient interface:**
```python
class MCPClient:
    def call_tool(
        self,
        server_name: str,    # must exist in registry.yaml
        tool_name: str,      # must be in server's declared tools list
        arguments: dict,
        timeout_seconds: int = 30
    ) -> MCPResult: ...

class MCPResult:
    server: str
    tool: str
    status: "success" | "error" | "timeout"
    data: dict | None
    error_message: str | None
    duration_ms: int
```

**Rules:**
- Skills call `MCPClient.call_tool()` — never raw HTTP to tool endpoints
- `server_name` must match an entry in `registry.yaml`
- MCP servers must respond within `timeout_seconds` or MCPResult returns `status=timeout`
- All MCP calls are logged automatically by MCPClient (for audit trail)

---

## LLM Adapter Contract

The interface between any component needing LLM reasoning and the configured LLM provider.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class LLMMessage:
    role: str  # "system" | "user" | "assistant"
    content: str

@dataclass
class LLMRequest:
    messages: list[LLMMessage]
    max_tokens: int = 1000
    temperature: float = 0.2
    tools: list[dict] | None = None  # optional tool definitions

@dataclass
class LLMResponse:
    content: str
    tool_calls: list[dict] | None = None
    input_tokens: int = 0
    output_tokens: int = 0
    model_used: str = ""

class LLMAdapter(ABC):
    @abstractmethod
    def complete(self, request: LLMRequest) -> LLMResponse: ...

    @abstractmethod
    def health_check(self) -> bool: ...
```

**Environment configuration:**
```bash
LLM_PROVIDER=azure_openai          # or: openai, bedrock, ollama, custom
LLM_MODEL=gpt-4o                   # model name within the provider
LLM_API_BASE=https://org-oai.azure.com
LLM_API_KEY=${SECRET_FROM_VAULT}
LLM_API_VERSION=2024-02-01         # Azure-specific
```

**Switching LLMs requires zero code changes** — only environment variable updates.

---

## MCP Server Self-Description Contract

Every MCP server built in this project must implement the `/describe` endpoint for registry auto-discovery.

```json
GET /describe

{
  "name": "splunk-mcp",
  "version": "1.0.0",
  "description": "MCP server for Splunk log search and alert management",
  "type": "observability",
  "tools": [
    {
      "name": "search_logs",
      "description": "Search Splunk logs by query, service, and time range",
      "input_schema": { "...json schema..." },
      "output_schema": { "...json schema..." }
    }
  ],
  "health": "/health",
  "deployment": "on-prem"
}
```

This contract enables the registry to validate MCP servers at startup and supports future auto-registration with an org Agent Provider platform.
