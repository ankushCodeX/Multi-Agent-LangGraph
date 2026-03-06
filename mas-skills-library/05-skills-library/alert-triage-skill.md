# Skill: Skills Library — Alert Triage Skill
**Phase:** 4-B | **Type:** Granular Logic | **Depends on:** 04-A (base-skill)

---

## Context
First skill called by DevOpsAgent. Classifies an incoming alert into severity
and category. Uses AppDynamics MCP server for context enrichment.
Pure LLM classification if MCP unavailable (graceful degradation).

---

## Parameters

| Parameter | Description |
|---|---|
| `{{APPDYNAMICS_BASE_URL}}` | AppDynamics controller URL |

---

## Dependencies
- `04-A` BaseSkill, SkillInput, SkillOutput
- AppDynamics MCP server must be running (or mocked)

---

## Prompt

```
Generate the file: skills-library/observability/alert_triage_skill.py

Create AlertTriageSkill(BaseSkill).

1. AlertTriageInput(SkillInput)
   Fields:
   - raw_input (str) — the raw alert text or webhook payload
   - source_system (str, default="unknown") — e.g. "appdynamics", "splunk"

2. AlertTriageOutput(SkillOutput)
   Fields:
   - severity (Literal["low","medium","high","critical"])
   - category (str) — e.g. "cpu_spike", "memory_leak", "latency", "error_rate"
   - summary (str) — 1-2 sentence human readable summary
   - affected_service (str | None)
   - confidence (float) — 0.0 to 1.0

3. AlertTriageSkill(BaseSkill)
   get_skill_name() → "alert_triage"
   get_mcp_server_name() → "appdynamics-mcp"

   async execute(input: AlertTriageInput) -> AlertTriageOutput:

   Step 1: If mcp_client is available:
     Call _call_mcp("get_alert_context", {"alert_text": input.raw_input})
     Returns: {"app_name": str, "tier": str, "node": str, "recent_events": list}
     Store as context_data

   Step 2: Build classification prompt
   System: "You are an alert triage expert. Classify the alert strictly.
            Respond ONLY in JSON: {severity, category, summary, affected_service, confidence}"
   User: f"Alert: {input.raw_input}\nContext: {context_data or 'none'}"

   Step 3: Call self.llm_provider.complete() if available,
   else use rule-based fallback:
   - If "critical" or "down" in raw_input.lower() → severity="critical"
   - If "warning" or "high" in raw_input.lower() → severity="high"
   - else → severity="medium"
   - category="unknown", confidence=0.5

   Step 4: Parse JSON response into AlertTriageOutput
   On JSON parse error: return medium severity with confidence=0.3

   Note: __init__ should accept optional llm_provider parameter too
   (agents inject both mcp_client and llm_provider)

Rules:
- structlog
- Never log raw_input content (may contain PII or sensitive system data)
- Full type annotations
- Rule-based fallback ensures skill works even without LLM
```

---

## Expected Output
- `skills-library/observability/alert_triage_skill.py` ~100 lines
- Works standalone without MCP or LLM (rule-based fallback)

## Verification
```python
import asyncio
from skills_library.observability.alert_triage_skill import AlertTriageSkill, AlertTriageInput
skill = AlertTriageSkill()  # no mcp_client
result = asyncio.run(skill.execute(AlertTriageInput(raw_input="CRITICAL: DB down")))
print(result.severity)  # → "critical"
```
