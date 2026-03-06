# Skill: Observability — OpenTelemetry + Structured Logging
**Phase:** 8-A/B | **Type:** High-level Structure | **Depends on:** All phases complete

---

## Context
Adds end-to-end distributed tracing (OpenTelemetry) and Splunk-compatible
structured logging (structlog). Traces flow from API gateway → orchestrator
→ agent → skill → MCP server. Every component gets a trace_id for correlation.

---

## Parameters

| Parameter | Description |
|---|---|
| `{{PROJECT_NAME}}` | Service name in traces |
| `{{SPLUNK_BASE_URL}}` | Splunk HEC endpoint for log export |

---

## Dependencies
- All previous phases complete
- `requirements.txt` has `opentelemetry-sdk`, `opentelemetry-exporter-otlp`

---

## Prompt

```
Generate observability setup for the entire multi-agent system.

FILE 1: orchestrator/core/telemetry.py

OTel setup module:

  setup_tracing(service_name: str, otlp_endpoint: str | None = None) -> TracerProvider:
    Creates TracerProvider with service.name resource attribute
    If otlp_endpoint provided: adds OTLPSpanExporter
    Else: adds ConsoleSpanExporter (for local dev)
    Sets as global TracerProvider
    Returns provider

  get_tracer(name: str) -> Tracer:
    Returns opentelemetry.trace.get_tracer(name)

  trace_async(span_name: str, attributes: dict = {}):
    Decorator for async functions. Wraps in a span.
    On exception: records exception in span, re-raises
    Usage: @trace_async("skill.alert_triage", {"skill.name": "alert_triage"})

  inject_trace_context(headers: dict) -> dict:
    Injects W3C trace context into an outgoing headers dict
    For passing trace context to MCP servers over HTTP

FILE 2: orchestrator/core/logging_config.py

structlog configuration for the whole Python codebase:

  configure_logging(log_level: str = "INFO", json_output: bool = True):
    Configure structlog processors:
    - Add timestamp (ISO8601, UTC)
    - Add log_level
    - Add service_name from env var
    - Add trace_id from current OTel span (if active)
    - JSONRenderer if json_output=True (Splunk-compatible)
    - ConsoleRenderer if json_output=False (local dev readable)
    Call once at application startup

  Splunk-compatible JSON format:
    {
      "timestamp": "2024-01-01T00:00:00Z",
      "level": "info",
      "service": "mas-orchestrator",
      "trace_id": "abc123",
      "event": "skill_executed",
      "skill_name": "alert_triage",
      "duration_ms": 145
    }

FILE 3: mcp-servers/shared/logger.ts (shared across all MCP servers)

Winston logger for TypeScript MCP servers:
  - JSON format (Splunk-compatible, same fields as Python logger)
  - Reads LOG_LEVEL from env
  - Adds service field from package.json name
  - Adds trace_id from W3C traceparent header if available
  - createLogger(serviceName: string) -> Logger factory function

FILE 4: orchestrator/core/metrics.py

Simple in-memory metrics (Prometheus-ready interface):
  MetricsCollector singleton:
    increment(counter_name: str, labels: dict = {}) -> None
    observe(histogram_name: str, value: float, labels: dict = {}) -> None
    get_snapshot() -> dict  (for /metrics endpoint)

  Pre-defined counters:
    tasks_total, tasks_failed, skills_executed, skills_failed,
    llm_calls_total, llm_calls_failed, mcp_calls_total, mcp_calls_failed

  Each counter keyed by labels dict for filtering

FILE 5: docker-compose.yml (project root)

Local development docker-compose:
  Services:
  - redis: redis:7-alpine, port 6379
  - orchestrator: builds from ./orchestrator, port 8080,
    env_file: .env, depends_on: redis
  - appdynamics-mcp: builds from ./mcp-servers/appdynamics-mcp,
    env_file: .env.appdynamics
  - splunk-mcp: (same pattern)
  - jaeger: jaegertracing/all-in-one:latest, ports 16686 (UI), 4317 (OTLP)

  Networks: all services on "mas-network" bridge
  Volumes: redis data persistence

Rules:
- trace_id MUST appear in every log line when a span is active
- Never log sensitive data in OTel attributes (mark sensitive spans as excluded)
- JSON log format must be parseable by Splunk HEC without transformation
- docker-compose uses env_file (never hardcoded credentials)
```

---

## Expected Output
- `orchestrator/core/telemetry.py`
- `orchestrator/core/logging_config.py`
- `orchestrator/core/metrics.py`
- `mcp-servers/shared/logger.ts`
- `docker-compose.yml`

## Verification
```bash
# Start local stack
docker-compose up -d redis jaeger

# Run orchestrator with tracing
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 \
LLM_PROVIDER=mock \
uvicorn api-gateway.main:app --port 8080

# Send test task
curl -X POST http://localhost:8080/v1/task \
  -H "Content-Type: application/json" \
  -d '{"raw_input":"test alert","source":"test"}'

# Check Jaeger UI for traces
open http://localhost:16686
```
