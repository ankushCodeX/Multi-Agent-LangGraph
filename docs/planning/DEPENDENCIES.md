# Dependencies — Nexus Validation List

## How to Use This Document

1. Take each package listed below to your Nexus repository administrator
2. Confirm availability and exact version in your internal Nexus mirror
3. Check the box and note the confirmed version
4. If unavailable, escalate before the phase begins (2-week lead time recommended)

**Nexus PyPI proxy URL:** *(fill in your org's Nexus PyPI URL)*  
**Nexus npm proxy URL:** *(fill in your org's Nexus npm URL)*

---

## Phase 0 — Foundation (Required Before Anything Else)

### Python Packages

| Package | Version (minimum) | Purpose | Nexus Confirmed? | Notes |
|---|---|---|---|---|
| `python` | 3.11+ | Runtime | ☐ | |
| `pydantic` | 2.5+ | Data contracts / validation | ☐ | Core to all contracts |
| `pydantic-settings` | 2.1+ | Config from env vars | ☐ | |
| `fastapi` | 0.109+ | API Gateway | ☐ | |
| `uvicorn` | 0.27+ | ASGI server for FastAPI | ☐ | |
| `python-dotenv` | 1.0+ | Local env file loading | ☐ | Dev only |
| `redis` | 5.0+ | Redis client | ☐ | State management |
| `httpx` | 0.26+ | Async HTTP client | ☐ | Used by MCPClient |
| `tenacity` | 8.2+ | Retry logic with backoff | ☐ | MCP call retries |
| `structlog` | 24.1+ | Structured logging | ☐ | |
| `pytest` | 7.4+ | Test runner | ☐ | |
| `pytest-asyncio` | 0.23+ | Async test support | ☐ | |
| `pytest-cov` | 4.1+ | Coverage reporting | ☐ | |
| `respx` | 0.20+ | Mock httpx calls in tests | ☐ | |
| `mypy` | 1.8+ | Static type checking | ☐ | |
| `ruff` | 0.2+ | Linting | ☐ | |

### Node.js Packages (for MCP servers in TypeScript)

| Package | Version | Purpose | Nexus Confirmed? | Notes |
|---|---|---|---|---|
| `node` | 20+ | Runtime | ☐ | |
| `@modelcontextprotocol/sdk` | latest stable | MCP server SDK | ☐ | Core MCP library |
| `zod` | 3.22+ | Schema validation | ☐ | |
| `typescript` | 5.3+ | TypeScript compiler | ☐ | |
| `ts-node` | 10.9+ | TS execution | ☐ | Dev only |
| `jest` | 29+ | Test runner | ☐ | |
| `@types/node` | 20+ | Node type definitions | ☐ | |
| `dotenv` | 16+ | Env file loading | ☐ | |
| `axios` | 1.6+ | HTTP client for MCP servers | ☐ | |

---

## Phase 1 — Observability (P1)

### Python Packages (additional)

| Package | Version | Purpose | Nexus Confirmed? | Notes |
|---|---|---|---|---|
| `langgraph` | 0.1+ | Orchestration framework | ☐ | **Critical — verify early** |
| `langchain-core` | 0.1+ | LangGraph dependency | ☐ | |
| `openai` | 1.12+ | OpenAI / Azure OpenAI adapter | ☐ | If org uses Azure OAI |
| `boto3` | 1.34+ | AWS Bedrock adapter | ☐ | If org uses Bedrock |
| `anthropic` | 0.20+ | Anthropic Claude adapter | ☐ | If org approves Claude |
| `pytest-httpx` | 0.28+ | Mock HTTP in tests | ☐ | For MCP server tests |

### Node.js Packages (additional)

| Package | Version | Purpose | Nexus Confirmed? | Notes |
|---|---|---|---|---|
| `nock` | 13+ | HTTP mocking in tests | ☐ | For MCP server unit tests |

---

## Phase 2 — Workflow (P2)

### Python Packages (additional)

| Package | Version | Purpose | Nexus Confirmed? | Notes |
|---|---|---|---|---|
| `apscheduler` | 3.10+ | Escalation timer scheduling | ☐ | For stale task escalation |
| `celery` | 5.3+ | Async task queue (optional) | ☐ | Only if async escalation needed |
| `redis` | already listed | Celery broker (if used) | ☐ | |

---

## Phase 3 — Knowledge Retrieval (P3)

### Python Packages (additional)

| Package | Version | Purpose | Nexus Confirmed? | Notes |
|---|---|---|---|---|
| `beautifulsoup4` | 4.12+ | HTML parsing for Confluence pages | ☐ | |
| `lxml` | 5.1+ | Fast XML/HTML parser | ☐ | BeautifulSoup backend |
| `markdownify` | 0.12+ | Convert HTML to Markdown | ☐ | For doc summarization |

---

## Phase 5 — Hardening

### Python Packages (additional)

| Package | Version | Purpose | Nexus Confirmed? | Notes |
|---|---|---|---|---|
| `prometheus-client` | 0.20+ | Metrics export | ☐ | For monitoring |
| `opentelemetry-sdk` | 1.22+ | Distributed tracing | ☐ | |
| `opentelemetry-exporter-otlp` | 1.22+ | OTLP trace export | ☐ | |
| `locust` | 2.23+ | Load testing | ☐ | Phase 5 only |

---

## Explicitly NOT Required (No External Calls)

These packages are intentionally excluded because they would require external network access or are not approved:

| Package | Reason Excluded |
|---|---|
| `langchain` (full) | Too many external dependencies — using `langchain-core` only |
| `chromadb` | Vector store — not needed in Phase 1, evaluate later |
| `sentence-transformers` | Requires model download — evaluate if local models approved |
| Any `transformers` / `torch` | Too heavy, requires model downloads |

---

## LLM Provider: What to Confirm with Org Model Directory

The system is model-agnostic. You need to confirm **one** of the following with your org's Model Directory team:

| Provider | Package Needed | Config Needed |
|---|---|---|
| Azure OpenAI | `openai>=1.12` | `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_DEPLOYMENT` |
| AWS Bedrock | `boto3>=1.34` | AWS credentials, region, model ID |
| On-prem Ollama | `httpx` (already listed) | `OLLAMA_BASE_URL`, model name |
| Anthropic Claude | `anthropic>=0.20` | `ANTHROPIC_API_KEY` |
| Custom OpenAI-compatible | `openai>=1.12` | `OPENAI_BASE_URL`, `OPENAI_API_KEY` |

**Action item:** Confirm with Org Model Directory team which LLM endpoint is available and get the required credentials and package approved.

---

## Version Pinning Policy

- All production dependencies must be **pinned to exact versions** in `requirements.txt` / `package.json`
- `requirements.txt` will use `==` (e.g. `pydantic==2.6.1`)
- `package.json` will use exact versions (e.g. `"@modelcontextprotocol/sdk": "0.6.0"`)
- Version bumps require a PR review and re-validation against Nexus
- `requirements-dev.txt` is separate from `requirements.txt` — dev tools never go to production
