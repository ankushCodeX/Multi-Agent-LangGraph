# Skill: Agents — DevOps Agent
**Phase:** 3-B | **Type:** Granular Logic | **Depends on:** 03-A (base-agent)

---

## Context
The highest-priority agent. Handles alert triage, log analysis, metric correlation,
and auto-incident creation. Orchestrates multiple observability skills in sequence.
This is the first agent built in Phase 2 of the project.

---

## Parameters
None — skill names are string constants defined in this file.

---

## Dependencies
- `03-A` BaseAgent, AgentError
- `02-E` OrchestratorState, AgentResult, AgentType, TaskPriority

---

## Prompt

```
Generate the file: orchestrator/agents/devops_agent.py

Create DevOpsAgent(BaseAgent) for observability and incident management.

SKILL_NAMES constants (top of file, so they can be imported by tests):
SKILL_ALERT_TRIAGE = "alert_triage"
SKILL_LOG_ANALYSIS = "log_analysis"
SKILL_METRIC_CORRELATION = "metric_correlation"
SKILL_INCIDENT_CREATOR = "incident_creator"
SKILL_RUNBOOK_LOOKUP = "runbook_lookup"

1. DevOpsAgent(BaseAgent)
   get_agent_type() -> AgentType.DEVOPS

   async run(state: OrchestratorState) -> AgentResult:
   Execute this sequence:

   Step 1 — Alert Triage
   Call _call_skill(SKILL_ALERT_TRIAGE, raw_input=state.task.raw_input)
   Result contains: severity (str), category (str), summary (str)

   Step 2 — Parallel enrichment (run concurrently with asyncio.gather)
   a. _call_skill(SKILL_LOG_ANALYSIS, summary=triage_result["summary"],
                  time_window_minutes=30)
   b. _call_skill(SKILL_METRIC_CORRELATION, category=triage_result["category"])
   Both may raise AgentError — catch individually, log warning, continue with empty result

   Step 3 — LLM synthesis
   Build a prompt combining triage + log + metric results
   Call _call_llm() to produce a human-readable incident summary (max 200 words)

   Step 4 — Conditional incident creation
   If severity in ["high", "critical"] OR state.task.priority == TaskPriority.CRITICAL:
     Call _call_skill(SKILL_INCIDENT_CREATOR, summary=llm_summary,
                      severity=triage_result["severity"])

   Step 5 — Runbook lookup
   Call _call_skill(SKILL_RUNBOOK_LOOKUP, category=triage_result["category"])
   Append runbook URL to final output if found

   Build and return AgentResult:
   - output = formatted string with all findings + runbook reference
   - skills_used = list of skills actually called (not failed ones)
   - success = True (even if enrichment steps partially failed)

   If Step 1 (triage) fails: return failed AgentResult immediately

Rules:
- asyncio.gather for parallel steps
- Track start_time = time.monotonic() for duration_ms
- structlog bound with task_id from state
- Full type annotations
```

---

## Expected Output
- `orchestrator/agents/devops_agent.py` ~120 lines
- Parallel skill calls for performance
- Graceful degradation when enrichment skills fail

## Verification
```python
# With mock skill registry
from orchestrator.agents.devops_agent import DevOpsAgent, SKILL_ALERT_TRIAGE
print(SKILL_ALERT_TRIAGE)  # → "alert_triage"
```
