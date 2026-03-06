# Skill: Auth — LDAP Module & Token Manager
**Phase:** 7-A/B | **Type:** Granular Logic | **Depends on:** 06-A (fastapi-main)
**⚠️ Production Phase — activate in Phase 4 of project plan only**

---

## Context
Two-part auth implementation:
Part A — LDAP connector that authenticates the service account against org directory.
Part B — Token manager that exchanges the service account credentials for a
Group AI Platform JWT token and handles refresh automatically.

---

## Parameters

| Parameter | Description |
|---|---|
| `{{SERVICE_ACCOUNT_ID}}` | LDAP service account username |
| `{{LLM_PROVIDER_ENDPOINT}}` | Group AI Platform base URL |
| `{{LLM_API_KEY_ENV_VAR}}` | Env var holding service account password |

---

## Dependencies
- `06-A` FastAPI app exists
- `requirements.txt` has `python-ldap`, `python-jose`, `passlib`

---

## Prompt

```
Generate two files for the production auth layer:

FILE 1: api-gateway/auth/ldap_connector.py

Create LDAPConnector for service account authentication.

Config (read from environment, never hardcoded):
  LDAP_SERVER_URL — e.g. ldap://ldap.internal.org:389
  LDAP_BASE_DN — e.g. dc=internal,dc=org
  LDAP_SERVICE_ACCOUNT_DN — full DN of service account
  LDAP_SERVICE_ACCOUNT_PASSWORD_ENV — env var name holding password (not the value)

LDAPConnector class:
  __init__(self): reads config, initializes ldap3 (NOT python-ldap — use ldap3
  which is pure Python and easier to mock; update requirements.txt accordingly)

  async authenticate_service_account() -> bool:
    Bind to LDAP using service account DN + password from env
    Return True if bind succeeds, False if credentials invalid
    Raise LDAPConnectionError if server unreachable (retryable)
    Log: server_url, account_id (not password), result
    Never log the password in any code path

  async get_service_account_groups() -> list[str]:
    After successful bind, search for group membership
    Returns list of group DNs the service account belongs to
    Used to verify the account has correct permissions

  LDAPConnectionError(Exception): message, server_url, retryable=True
  LDAPAuthError(Exception): message, account_id

FILE 2: api-gateway/auth/token_manager.py

Create TokenManager that fetches and refreshes Group AI Platform JWT tokens.

TokenManager class:
  __init__(self, ldap_connector: LDAPConnector):
    Stores connector
    Initializes _cached_token: str | None = None
    Initializes _token_expiry: datetime | None = None
    asyncio.Lock for thread-safe refresh

  async get_token() -> str:
    If cached token is valid (expiry > now + 60s buffer): return cached
    Else: call _refresh_token()
    Return token

  async _refresh_token() -> None:
    Acquire lock (prevent concurrent refreshes)
    1. Authenticate service account via ldap_connector
    2. POST to {{LLM_PROVIDER_ENDPOINT}}/auth/token with service account credentials
       Body: {"grant_type": "service_account", "account_id": "{{SERVICE_ACCOUNT_ID}}",
               "credential": <from env var {{LLM_API_KEY_ENV_VAR}}>}
    3. Parse response: access_token, expires_in (seconds)
    4. Set _cached_token, _token_expiry
    5. Log: token refreshed, expires_at (not the token value)
    On failure: raise TokenRefreshError with is_retryable bool

  TokenRefreshError(Exception): message, is_retryable

FILE 3: api-gateway/auth/auth_middleware.py

FastAPI middleware that injects the token into every LLM provider call.

  TokenInjectionMiddleware:
    On startup: initialize TokenManager, inject into app.state.token_manager
    Override OrgGroupAIProvider._get_api_key() to call token_manager.get_token()
    instead of reading env var directly (monkey-patch is acceptable for Phase 4
    — clean interface in Phase 5)

FILE 4: api-gateway/auth/tests/test_auth.py
  Tests:
  - test_ldap_successful_bind (mock ldap3.Connection)
  - test_ldap_wrong_password_returns_false (not raises)
  - test_ldap_server_unreachable_raises_connection_error
  - test_token_manager_caches_token (calls _refresh_token only once for 2 get_token calls)
  - test_token_manager_refreshes_on_expiry
  - test_token_never_logged (caplog assertion)

Rules:
- ldap3 library instead of python-ldap (pure Python, mockable)
- asyncio.Lock for token refresh concurrency
- NEVER log token values, passwords, or credentials
- Add ldap3==3.4.3 and update requirements.txt
```

---

## Expected Output
- `api-gateway/auth/ldap_connector.py`
- `api-gateway/auth/token_manager.py`
- `api-gateway/auth/auth_middleware.py`
- `api-gateway/auth/tests/test_auth.py` with 6 tests

## Verification
```bash
# All tests with mocked LDAP (no real LDAP server needed)
pytest api-gateway/auth/tests/ -v
# Expected: 6 passed
```
