# Skill: LLM Abstraction — Org Group AI Provider
**Phase:** 1-B | **Type:** Granular Logic | **Depends on:** 01-A (base-provider)

---

## Context
The PRIMARY provider implementation targeting your organization's Group AI Platform.
This is what runs in production. Implements the BaseLLMProvider contract using
your org's approved LLM endpoint (OpenAI-compatible API format assumed — adjust
prompt if org uses a different wire format).

---

## Parameters

| Parameter | Description |
|---|---|
| `{{LLM_PROVIDER_ENDPOINT}}` | Group AI Platform base URL e.g. `https://ai-platform.internal.org/v1` |
| `{{LLM_MODEL_NAME}}` | Approved model identifier e.g. `gpt-4o` or org-specific name |
| `{{LLM_API_KEY_ENV_VAR}}` | Env var name e.g. `GROUP_AI_API_KEY` |

---

## Dependencies
- `01-A` base-provider.py exists with `BaseLLMProvider`, `LLMMessage`, `LLMResponse`, `LLMConfig`

---

## Prompt

```
Generate the file: orchestrator/core/providers/org_group_ai.py

Implement OrgGroupAIProvider(BaseLLMProvider) for the organization's Group AI Platform.

The provider must:

1. Call {{LLM_PROVIDER_ENDPOINT}}/chat/completions using httpx.AsyncClient
   (OpenAI-compatible wire format — messages array, model, max_tokens, temperature)

2. Default config values baked in as class-level constants:
   DEFAULT_ENDPOINT = "{{LLM_PROVIDER_ENDPOINT}}"
   DEFAULT_MODEL = "{{LLM_MODEL_NAME}}"
   DEFAULT_API_KEY_ENV_VAR = "{{LLM_API_KEY_ENV_VAR}}"

3. __init__(self, config: LLMConfig | None = None)
   If config is None, build one from the defaults above + env vars
   Never hardcode credentials

4. async complete(messages, **kwargs) -> LLMResponse
   - Build Authorization: Bearer header from self._get_api_key()
   - Add header X-Request-Source: "mas-platform" for org audit trail
   - POST to endpoint with timeout from config
   - Parse response into LLMResponse
   - Use the retry logic inherited from BaseLLMProvider
   - On 401/403: raise LLMProviderError with clear message (auth failure)
   - On 429: raise LLMProviderError with message "rate_limited"
   - On 5xx: let tenacity retry, then raise LLMProviderError

5. async health_check() -> bool
   GET {{LLM_PROVIDER_ENDPOINT}}/models — return True if 200, False otherwise
   Catch all exceptions, log warning, return False (never raise)

6. get_provider_name() -> str
   Return "org_group_ai"

7. Private method _parse_response(raw: dict) -> LLMResponse
   Extract content, model, usage, finish_reason safely using .get()
   Never assume fields exist — default gracefully

Rules:
- Full type annotations
- structlog logging — log request metadata (model, token counts) but NEVER log
  the message content or API key
- httpx.AsyncClient must use context manager (async with)
- All imports from orchestrator.core.llm_provider (the base module)
- Include a module-level __all__ = ["OrgGroupAIProvider"]
```

---

## Expected Output
- `orchestrator/core/providers/org_group_ai.py` ~100 lines
- Implements all 3 abstract methods from BaseLLMProvider
- No hardcoded credentials anywhere

## Verification
```python
import asyncio, os
os.environ["GROUP_AI_API_KEY"] = "test"
from orchestrator.core.providers.org_group_ai import OrgGroupAIProvider
p = OrgGroupAIProvider()
print(p.get_provider_name())  # → "org_group_ai"
# health_check() will return False (no real endpoint) — that's expected
```
