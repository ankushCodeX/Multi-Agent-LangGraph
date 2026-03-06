# Skill: LLM Abstraction — Base Provider
**Phase:** 1-A | **Type:** Granular Logic | **Depends on:** 00-A, 00-B

---

## Context
The single most critical architectural piece. This abstract base class is the
contract that ALL LLM providers must implement. Swapping models = swapping one
config value. Zero application code changes required.

---

## Parameters

| Parameter | Description |
|---|---|
| `{{PROJECT_NAME}}` | Root project name |
| `{{PYTHON_VERSION}}` | e.g. `3.11` |

---

## Dependencies
- `00-A` scaffold complete
- `00-B` requirements.txt has `pydantic`, `httpx`, `tenacity`

---

## Prompt

```
Generate the file: orchestrator/core/llm_provider.py

This is the model-agnostic LLM abstraction layer for a multi-agent system.
Use Python {{PYTHON_VERSION}}. Use only packages from the requirements.txt
(pydantic, httpx, tenacity — no langchain imports here).

Generate these classes and types in one file:

1. LLMMessage (Pydantic BaseModel)
   Fields: role (Literal["system","user","assistant"]), content (str)

2. LLMResponse (Pydantic BaseModel)
   Fields:
   - content (str)
   - model (str)
   - usage (dict with prompt_tokens, completion_tokens, total_tokens)
   - finish_reason (str)
   - raw (dict | None) — the raw provider response, for debugging

3. LLMConfig (Pydantic BaseModel)
   Fields:
   - provider_name (str)
   - endpoint (str)
   - model_name (str)
   - api_key_env_var (str) — name of the env var, never the value itself
   - max_tokens (int, default=2000)
   - temperature (float, default=0.1)
   - timeout_seconds (int, default=30)
   - max_retries (int, default=3)

4. BaseLLMProvider (ABC)
   Abstract methods:
   - async complete(messages: list[LLMMessage], **kwargs) -> LLMResponse
   - async health_check() -> bool
   - def get_provider_name() -> str

   Concrete methods (shared logic, not to be overridden):
   - _get_api_key() -> str
     Reads os.environ[self.config.api_key_env_var], raises clear error if missing
   - _build_retry_decorator()
     Uses tenacity: retry on httpx.TimeoutException and httpx.HTTPStatusError (5xx),
     stop after config.max_retries, wait exponential(min=1, max=10),
     log each retry attempt using structlog

5. LLMProviderError (Exception)
   With fields: provider_name, status_code (int | None), message

Rules:
- Full type annotations everywhere
- Docstrings on every class and method
- structlog for all logging (never print())
- API keys NEVER logged, even partially
- The file must be importable standalone with no side effects
```

---

## Expected Output
- `orchestrator/core/llm_provider.py` — single file, ~120 lines
- All 5 classes/types defined and complete
- No external imports beyond stdlib + pydantic + httpx + tenacity + structlog

## Verification
```python
# Quick smoke test
from orchestrator.core.llm_provider import (
    BaseLLMProvider, LLMMessage, LLMResponse, LLMConfig, LLMProviderError
)
# Should import cleanly with no errors
```
