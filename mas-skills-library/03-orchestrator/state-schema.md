# Skill: Orchestrator — Shared State Schema
**Phase:** 2-E | **Type:** Granular Logic | **Depends on:** 01-D

---

## Context
The Pydantic state schema that flows through the LangGraph state machine.
Every agent, router, and node reads from and writes to this shared object.
Defining it first prevents type mismatches across all later skills.

---

## Parameters

| Parameter | Description |
|---|---|
| `{{PROJECT_NAME}}` | Used in module docstring |

---

## Dependencies
- `01-A` LLMMessage, LLMResponse defined
- `requirements.txt` has `pydantic`, `langgraph`

---

## Prompt

```
Generate the file: orchestrator/graph/state.py

Define the complete shared state schema for a LangGraph multi-agent orchestrator.
Use Pydantic BaseModel throughout. Python 3.11+.

Classes to generate:

1. TaskPriority (str Enum)
   Values: LOW, MEDIUM, HIGH, CRITICAL

2. AgentType (str Enum)
   Values: DEVOPS, WORKFLOW, KNOWLEDGE, SDLC, UNKNOWN

3. TaskStatus (str Enum)
   Values: PENDING, ROUTING, IN_PROGRESS, AWAITING_TOOL, COMPLETED, FAILED

4. IncomingTask (Pydantic BaseModel)
   Fields:
   - task_id (str) — UUID, auto-generated default
   - raw_input (str) — the original user/system text
   - source (str) — where it came from e.g. "slack", "webhook", "api"
   - priority (TaskPriority, default=MEDIUM)
   - metadata (dict, default={}) — arbitrary k/v from the trigger

5. AgentResult (Pydantic BaseModel)
   Fields:
   - agent_type (AgentType)
   - output (str)
   - skills_used (list[str], default=[])
   - mcp_servers_called (list[str], default=[])
   - success (bool)
   - error_message (str | None)
   - duration_ms (int | None)

6. OrchestratorState (Pydantic BaseModel) — the main graph state
   Fields:
   - task (IncomingTask)
   - intent (str | None) — filled by intent classifier node
   - intent_confidence (float | None) — 0.0 to 1.0
   - assigned_agent (AgentType, default=UNKNOWN)
   - decomposed_steps (list[str], default=[]) — from task decomposer
   - agent_results (list[AgentResult], default=[])
   - final_response (str | None)
   - status (TaskStatus, default=PENDING)
   - created_at (datetime, auto UTC now)
   - updated_at (datetime, auto UTC now)
   - error (str | None)

   Methods:
   - def add_result(self, result: AgentResult) -> None
     Appends to agent_results, updates updated_at
   - def mark_failed(self, error: str) -> None
     Sets status=FAILED, error=error, updated_at=now
   - def to_summary_dict(self) -> dict
     Returns a minimal dict: task_id, intent, assigned_agent, status, error
     (for logging — never includes raw_input to avoid leaking sensitive data)

Rules:
- All datetime fields use timezone-aware UTC
- No mutable default arguments (use Field(default_factory=...))
- Full docstrings
- __all__ exports all 6 classes
```

---

## Expected Output
- `orchestrator/graph/state.py` ~100 lines
- All enums and models importable

## Verification
```python
from orchestrator.graph.state import OrchestratorState, IncomingTask, AgentType
t = IncomingTask(raw_input="test alert", source="webhook")
s = OrchestratorState(task=t)
print(s.status)  # TaskStatus.PENDING
print(s.to_summary_dict())
```
