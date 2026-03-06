# Skill: LLM Abstraction — Provider Tests
**Phase:** 1-E | **Type:** Granular Logic | **Depends on:** 01-D

---

## Context
Proves the model-agnostic contract works. Tests that any provider can be swapped
via config, that auth errors surface cleanly, and that retries work correctly.
Uses no real LLM calls — fully mockable.

---

## Parameters
None — tests use MockLLMProvider and httpx mock.

---

## Dependencies
- `01-A` through `01-D` all complete
- `requirements.txt` has `pytest`, `pytest-asyncio`, `pytest-mock`

---

## Prompt

```
Generate the file: orchestrator/tests/test_llm_providers.py

Write pytest tests for the LLM abstraction layer. Use pytest-asyncio for async tests.
Use pytest-mock and httpx's MockTransport / httpx.MockResponse for HTTP mocking.
No real HTTP calls in any test.

Test cases to implement:

1. test_mock_provider_returns_response
   - create MockLLMProvider via factory (LLM_PROVIDER=mock)
   - call complete() with 2 messages
   - assert LLMResponse.content starts with "MOCK_RESPONSE"
   - assert LLMResponse.model == "mock-model"

2. test_mock_provider_health_check
   - MockLLMProvider.health_check() returns True

3. test_factory_unknown_provider_raises
   - LLMProviderFactory.create("nonexistent") raises ValueError
   - error message contains the list of valid providers

4. test_factory_env_var_controls_provider
   - set LLM_PROVIDER env var to "mock"
   - get_llm_provider() returns MockLLMProvider instance

5. test_org_provider_builds_correct_headers (mock HTTP)
   - Patch httpx.AsyncClient.post to capture the request
   - Call OrgGroupAIProvider.complete() with a test message
   - Assert Authorization header = "Bearer <key from env>"
   - Assert X-Request-Source header = "mas-platform"

6. test_org_provider_401_raises_provider_error (mock HTTP)
   - Mock httpx to return 401
   - Assert LLMProviderError is raised
   - Assert error message contains "auth"

7. test_org_provider_never_logs_api_key (mock HTTP + caplog)
   - Call OrgGroupAIProvider with LLM_API_KEY_ENV_VAR set
   - Assert the API key value never appears in any log record

8. test_api_key_missing_raises_clear_error
   - Ensure env var is NOT set
   - Instantiate OrgGroupAIProvider
   - Call complete() — assert raises LLMProviderError with clear message

Rules:
- Each test is independent (no shared state)
- Use pytest fixtures for repeated setup
- All tests must pass with LLM_PROVIDER=mock (no org network needed)
- Add a conftest.py in orchestrator/tests/ that sets LLM_PROVIDER=mock by default
```

---

## Expected Output
- `orchestrator/tests/test_llm_providers.py` with all 8 tests
- `orchestrator/tests/conftest.py` with base fixtures
- All tests pass with `pytest orchestrator/tests/test_llm_providers.py`

## Verification
```bash
cd orchestrator
LLM_PROVIDER=mock pytest tests/test_llm_providers.py -v
# Expected: 8 passed
```
