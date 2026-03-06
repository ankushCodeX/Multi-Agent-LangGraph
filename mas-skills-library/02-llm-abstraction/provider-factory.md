# Skill: LLM Abstraction — Provider Factory
**Phase:** 1-D | **Type:** Granular Logic | **Depends on:** 01-B, 01-C

---

## Context
Config-driven factory that returns the right LLM provider based on environment.
This is the ONLY place the application decides which LLM to use.
Changing `LLM_PROVIDER=azure` in `.env` is all that's needed to swap models.

---

## Parameters

| Parameter | Description |
|---|---|
| `{{LLM_PROVIDER_ENDPOINT}}` | Default org endpoint |
| `{{LLM_MODEL_NAME}}` | Default org model name |
| `{{LLM_API_KEY_ENV_VAR}}` | Default env var for API key |

---

## Dependencies
- `01-A` BaseLLMProvider
- `01-B` OrgGroupAIProvider
- `01-C` AzureOpenAIProvider

---

## Prompt

```
Generate the file: orchestrator/core/providers/factory.py

Create LLMProviderFactory with these requirements:

1. PROVIDER_REGISTRY: dict mapping string keys to provider classes
   Supported keys: "org_group_ai", "azure_openai", "mock"
   "mock" key maps to a MockLLMProvider (defined in this same file, for testing)

2. MockLLMProvider(BaseLLMProvider)
   For use in tests and local dev only.
   complete() returns a hardcoded LLMResponse with:
   - content = "MOCK_RESPONSE: " + str(len(messages)) + " messages received"
   - model = "mock-model"
   - usage = {"prompt_tokens": 0, "completion_tokens": 0, "total_tokens": 0}
   - finish_reason = "stop"
   health_check() always returns True
   get_provider_name() returns "mock"

3. LLMProviderFactory.create(provider_type: str | None = None) -> BaseLLMProvider
   - If provider_type is None, read env var LLM_PROVIDER (default: "org_group_ai")
   - Look up in PROVIDER_REGISTRY
   - If not found, raise ValueError with list of valid options
   - Instantiate and return the provider
   - Log which provider was instantiated (not the config details)

4. Module-level convenience function: get_llm_provider() -> BaseLLMProvider
   Calls LLMProviderFactory.create() — this is what all agents import

Rules:
- New providers can be added to PROVIDER_REGISTRY with zero other code changes
- structlog for logging
- Full type annotations
- __all__ = ["LLMProviderFactory", "get_llm_provider"]
```

---

## Expected Output
- `orchestrator/core/providers/factory.py` ~80 lines
- `MockLLMProvider` embedded in same file
- `get_llm_provider()` as single import point for rest of codebase

## Verification
```python
import os
os.environ["LLM_PROVIDER"] = "mock"
from orchestrator.core.providers.factory import get_llm_provider
p = get_llm_provider()
print(p.get_provider_name())  # → "mock"
```
