# Skill: Skills Library — Base Skill Contract
**Phase:** 4-A | **Type:** Granular Logic | **Depends on:** 03-A (base-agent)

---

## Context
The contract every skill must implement. Skills are pure-ish functions:
given typed input → call MCP server or internal logic → return typed output.
Skills are independently testable by mocking the MCP client.

---

## Parameters
None

---

## Dependencies
- `requirements.txt` has `pydantic`, `httpx`, `tenacity`

---

## Prompt

```
Generate the file: skills-library/base/base_skill.py

Create the base contract for all skills in the multi-agent system.

1. SkillInput (Pydantic BaseModel, base class)
   Fields: task_id (str | None) — for tracing only

2. SkillOutput (Pydantic BaseModel, base class)
   Fields:
   - success (bool)
   - error_message (str | None)
   - duration_ms (int | None)
   - mcp_server_used (str | None)

3. SkillError(Exception)
   Fields: skill_name (str), message (str), retryable (bool)

4. BaseSkill(ABC)
   __init__(self, mcp_client: Any | None = None)
   - Stores mcp_client (injected — enables test mocking)
   - structlog logger bound with skill_name=self.get_skill_name()

   Abstract methods:
   - async execute(input: SkillInput) -> SkillOutput
   - def get_skill_name() -> str
   - def get_mcp_server_name() -> str | None
     Return None if skill uses no MCP server (pure logic)

   Concrete methods:
   - async _call_mcp(tool_name: str, arguments: dict) -> dict
     Calls self.mcp_client.call_tool(tool_name, arguments)
     Raises SkillError(retryable=True) on timeout
     Raises SkillError(retryable=False) on auth error
     Logs tool_name and duration_ms (not arguments, may contain sensitive data)

   - def _time_execution(self) -> contextmanager
     Context manager that measures elapsed ms, stores in self._last_duration_ms

Rules:
- mcp_client: Any allows injecting mock in tests
- Full type annotations
- __all__ = ["BaseSkill", "SkillInput", "SkillOutput", "SkillError"]
```

---

## Expected Output
- `skills-library/base/base_skill.py` ~80 lines

## Verification
```python
from skills_library.base.base_skill import BaseSkill, SkillInput, SkillOutput, SkillError
# Clean import, BaseSkill not instantiable directly
```
