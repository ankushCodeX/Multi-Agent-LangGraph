# Skill: Agents — Base Agent
**Phase:** 3-A | **Type:** Granular Logic | **Depends on:** 02-A, 02-E

---

## Context
Abstract base that all specialist agents implement. Enforces the contract:
every agent receives the shared state, uses the skill library, and returns
a typed AgentResult. Agents never call LLM or MCP directly — they call skills.

---

## Parameters
None

---

## Dependencies
- `02-E` OrchestratorState, AgentResult, AgentType
- `01-A` BaseLLMProvider

---

## Prompt

```
Generate the file: orchestrator/agents/base_agent.py

Create BaseAgent(ABC) for the multi-agent orchestrator.

1. BaseAgent(ABC)
   __init__(self, llm_provider: BaseLLMProvider, skill_registry: dict[str, Any])
   - Stores llm_provider and skill_registry
   - Initializes structlog logger bound with agent_type=self.get_agent_type().value

   Abstract methods:
   - async run(state: OrchestratorState) -> AgentResult
   - def get_agent_type() -> AgentType

   Concrete helper methods:
   - async _call_skill(skill_name: str, **kwargs) -> dict
     Looks up skill_name in self.skill_registry
     Raises AgentError if not found (with message listing available skills)
     Calls skill.execute(**kwargs) and returns result dict
     Logs skill_name, duration_ms

   - async _call_llm(system_prompt: str, user_content: str) -> str
     Wraps llm_provider.complete() with standard message construction
     Returns just the content string
     Raises AgentError on LLMProviderError

   - def _build_result(output, skills_used, mcp_servers, success,
                       error=None, duration_ms=None) -> AgentResult
     Convenience constructor for AgentResult

2. AgentError(Exception)
   Fields: agent_type (str), message (str), skill_name (str | None)

Rules:
- structlog, full type annotations
- Never log state.task.raw_input — use state.task.task_id only
- __all__ = ["BaseAgent", "AgentError"]
```

---

## Expected Output
- `orchestrator/agents/base_agent.py` ~80 lines

## Verification
```python
from orchestrator.agents.base_agent import BaseAgent, AgentError
# Should import cleanly. BaseAgent cannot be instantiated (ABC).
```
