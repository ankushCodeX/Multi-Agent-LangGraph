# Skill: API Gateway — FastAPI Main
**Phase:** 6-A | **Type:** High-level Structure | **Depends on:** 02-A (graph-skeleton)

---

## Context
The HTTP entry point for the entire system. Receives tasks from webhooks, Slack,
and other callers. Routes them into the LangGraph orchestrator. Also serves as
the security boundary between on-prem and external systems.

---

## Parameters

| Parameter | Description |
|---|---|
| `{{PROJECT_NAME}}` | Used in app title |
| `{{LLM_PROVIDER_ENDPOINT}}` | For config display in /health |

---

## Dependencies
- `02-A` run_task() importable from orchestrator
- `01-D` get_llm_provider() available
- `requirements.txt` has `fastapi`, `uvicorn`, `pydantic`

---

## Prompt

```
Generate the file: api-gateway/main.py

Create a production-ready FastAPI application that wraps the LangGraph orchestrator.

1. App setup
   - FastAPI(title="{{PROJECT_NAME}} Gateway", version="1.0.0")
   - Startup event: initialize LLM provider, agent registry, build graph
     Store in app.state so routes can access them
   - Shutdown event: cleanup connections
   - CORS middleware: internal origins only (configurable via ALLOWED_ORIGINS env var)
   - Request ID middleware: adds X-Request-ID header to every response
   - Structured access logging middleware using structlog (log: method, path,
     status_code, duration_ms, request_id — never log request body)

2. Pydantic request/response models
   TaskRequest:
     raw_input (str, max_length=5000)
     source (str, default="api")
     priority (str, default="medium") — validated against TaskPriority enum
     metadata (dict, default={})

   TaskResponse:
     task_id (str)
     status (str)
     intent (str | None)
     assigned_agent (str | None)
     final_response (str | None)
     duration_ms (int)
     error (str | None)

3. Routes

   POST /v1/task
   - Validate TaskRequest
   - Call run_task() from orchestrator
   - Return TaskResponse
   - On exception: return 500 with error field set (never expose stack trace)
   - Add task_id to response headers for client tracing

   GET /health
   - Returns: { status: "ok"|"degraded", llm_provider: str,
                llm_healthy: bool, timestamp: str }
   - Calls llm_provider.health_check() — if False, status="degraded" (still 200)
   - Never returns 5xx from /health (monitoring tools interpret that as down)

   GET /ready
   - Returns 200 if app.state.graph is initialized, 503 if not
   - For Kubernetes readiness probe

   GET /metrics (placeholder)
   - Returns basic counters: tasks_total, tasks_failed, avg_duration_ms
   - Use a simple in-memory counter (real Prometheus in Phase 8)

4. Error handlers
   - RequestValidationError → 422 with field-level error details
   - Generic Exception → 500 with generic message (no internal details)

5. uvicorn entrypoint at bottom:
   if __name__ == "__main__": uvicorn.run(app, host="0.0.0.0", port=8080)

Rules:
- structlog throughout
- Never log raw_input in access logs
- App title, version from environment variables (override in tests)
- Full type annotations
```

---

## Expected Output
- `api-gateway/main.py` ~180 lines
- All routes, middleware, startup/shutdown

## Verification
```bash
cd api-gateway
LLM_PROVIDER=mock uvicorn main:app --port 8080 &
curl http://localhost:8080/health
# → {"status":"degraded","llm_healthy":false,...}  (mock provider)
curl http://localhost:8080/ready
# → 200 once startup completes
```
