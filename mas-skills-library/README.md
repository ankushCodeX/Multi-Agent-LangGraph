# Multi-Agent System — Copilot Skills Catalog

## How to Use This Library

Each `.md` file in this catalog is a **self-contained Copilot prompt**.
Feed it to your org's Copilot one at a time, replacing all `{{PARAMETER}}` placeholders
before submitting.

### Workflow
```
1. Open the skill file for the component you want to generate
2. Replace all {{PARAMETERS}} with your actual values
3. Paste into Copilot
4. Copilot generates the code/config
5. Verify output matches the "Expected Output" section of the skill
6. Move to next skill (respecting the dependency order)
```

### Parameter Reference (fill these in once, reuse everywhere)

| Parameter | Description | Your Value |
|---|---|---|
| `{{NEXUS_REGISTRY_URL}}` | Internal Nexus npm/PyPI URL | ___________ |
| `{{LLM_PROVIDER_ENDPOINT}}` | Group AI Platform base URL | ___________ |
| `{{LLM_MODEL_NAME}}` | Approved model name/ID | ___________ |
| `{{LLM_API_KEY_ENV_VAR}}` | Env var name holding the API key | ___________ |
| `{{APPDYNAMICS_BASE_URL}}` | AppDynamics controller URL | ___________ |
| `{{SPLUNK_BASE_URL}}` | Splunk API base URL | ___________ |
| `{{PAGERDUTY_BASE_URL}}` | PagerDuty API base URL | ___________ |
| `{{JIRA_BASE_URL}}` | Jira instance URL | ___________ |
| `{{CONFLUENCE_BASE_URL}}` | Confluence instance URL | ___________ |
| `{{SLACK_BASE_URL}}` | Slack API base URL | ___________ |
| `{{ORG_PACKAGE_PREFIX}}` | Internal package name prefix | ___________ |
| `{{SERVICE_ACCOUNT_ID}}` | LDAP service account username | ___________ |
| `{{PROJECT_NAME}}` | Root project name (snake_case) | ___________ |
| `{{PYTHON_VERSION}}` | Approved Python version | ___________ |
| `{{NODE_VERSION}}` | Approved Node.js version | ___________ |

---

## Skills Catalog — Execution Order

### PHASE 0 — Meta & Scaffolding
| # | Skill File | What It Generates | Dependency |
|---|---|---|---|
| 00-A | `00-meta/project-scaffold.md` | Full folder structure + .gitignore + base configs | None |
| 00-B | `00-meta/nexus-config.md` | `.npmrc`, `pip.conf`, `requirements-base.txt` | 00-A |
| 00-C | `00-meta/env-config.md` | `.env.example`, `config.py`, `settings.ts` | 00-A |

### PHASE 1 — LLM Abstraction Layer
| # | Skill File | What It Generates | Dependency |
|---|---|---|---|
| 01-A | `02-llm-abstraction/base-provider.md` | `BaseLLMProvider` abstract class (Python) | 00-A |
| 01-B | `02-llm-abstraction/org-provider.md` | `OrgGroupAIProvider` implementation | 01-A |
| 01-C | `02-llm-abstraction/azure-provider.md` | `AzureOpenAIProvider` fallback | 01-A |
| 01-D | `02-llm-abstraction/provider-factory.md` | `LLMProviderFactory` + config-driven swap | 01-B, 01-C |
| 01-E | `02-llm-abstraction/provider-tests.md` | Unit tests for all providers + mock | 01-D |

### PHASE 2 — Orchestrator
| # | Skill File | What It Generates | Dependency |
|---|---|---|---|
| 02-A | `03-orchestrator/graph-skeleton.md` | LangGraph state machine skeleton | 01-D |
| 02-B | `03-orchestrator/intent-classifier.md` | Intent classification node | 02-A |
| 02-C | `03-orchestrator/task-decomposer.md` | Task decomposition node | 02-B |
| 02-D | `03-orchestrator/agent-router.md` | Hybrid rules+LLM routing node | 02-C |
| 02-E | `03-orchestrator/state-schema.md` | Shared state schema (Pydantic) | 02-A |
| 02-F | `03-orchestrator/orchestrator-tests.md` | Integration tests for full graph | 02-D |

### PHASE 3 — Agents
| # | Skill File | What It Generates | Dependency |
|---|---|---|---|
| 03-A | `04-agents/base-agent.md` | `BaseAgent` abstract class | 02-A |
| 03-B | `04-agents/devops-agent.md` | `DevOpsAgent` (alert triage, observability) | 03-A |
| 03-C | `04-agents/workflow-agent.md` | `WorkflowAgent` (incident, escalation) | 03-A |
| 03-D | `04-agents/knowledge-agent.md` | `KnowledgeAgent` (runbook, confluence) | 03-A |
| 03-E | `04-agents/sdlc-agent.md` | `SDLCAgent` (PR, deploy, code) | 03-A |
| 03-F | `04-agents/agent-tests.md` | Unit tests per agent with mocked skills | 03-B,C,D,E |

### PHASE 4 — Skills Library
| # | Skill File | What It Generates | Dependency |
|---|---|---|---|
| 04-A | `05-skills-library/base-skill.md` | `BaseSkill` contract + input/output models | 03-A |
| 04-B | `05-skills-library/alert-triage-skill.md` | `AlertTriageSkill` | 04-A |
| 04-C | `05-skills-library/log-analysis-skill.md` | `LogAnalysisSkill` | 04-A |
| 04-D | `05-skills-library/metric-correlation-skill.md` | `MetricCorrelationSkill` | 04-A |
| 04-E | `05-skills-library/incident-creator-skill.md` | `IncidentCreatorSkill` | 04-A |
| 04-F | `05-skills-library/runbook-lookup-skill.md` | `RunbookLookupSkill` | 04-A |
| 04-G | `05-skills-library/task-router-skill.md` | `TaskRouterSkill` | 04-A |
| 04-H | `05-skills-library/confluence-search-skill.md` | `ConfluenceSearchSkill` | 04-A |
| 04-I | `05-skills-library/skills-tests.md` | Unit tests for all skills with mocks | 04-B→H |

### PHASE 5 — MCP Servers (TypeScript, each standalone)
| # | Skill File | What It Generates | Dependency |
|---|---|---|---|
| 05-A | `06-mcp-servers/mcp-server-template.md` | Base MCP server template (TS) | 00-B |
| 05-B | `06-mcp-servers/appdynamics-mcp.md` | AppDynamics MCP server | 05-A |
| 05-C | `06-mcp-servers/splunk-mcp.md` | Splunk MCP server | 05-A |
| 05-D | `06-mcp-servers/pagerduty-mcp.md` | PagerDuty MCP server | 05-A |
| 05-E | `06-mcp-servers/jira-mcp.md` | Jira MCP server | 05-A |
| 05-F | `06-mcp-servers/confluence-mcp.md` | Confluence MCP server | 05-A |
| 05-G | `06-mcp-servers/slack-mcp.md` | Slack MCP server | 05-A |
| 05-H | `06-mcp-servers/mcp-registry.md` | MCP registry config + health check | 05-B→G |
| 05-I | `06-mcp-servers/mcp-server-tests.md` | Jest tests per MCP server | 05-B→G |

### PHASE 6 — API Gateway
| # | Skill File | What It Generates | Dependency |
|---|---|---|---|
| 06-A | `07-api-gateway/fastapi-main.md` | FastAPI app entrypoint + middleware | 02-A |
| 06-B | `07-api-gateway/task-router-api.md` | `/v1/task` endpoint | 06-A |
| 06-C | `07-api-gateway/health-api.md` | `/health`, `/ready` endpoints | 06-A |
| 06-D | `07-api-gateway/gateway-tests.md` | API endpoint tests | 06-B, 06-C |

### PHASE 7 — Auth (Production)
| # | Skill File | What It Generates | Dependency |
|---|---|---|---|
| 07-A | `08-auth/ldap-module.md` | LDAP connector + service account auth | 06-A |
| 07-B | `08-auth/token-manager.md` | Token fetch/refresh for Group AI Platform | 07-A |
| 07-C | `08-auth/auth-middleware.md` | FastAPI auth middleware | 07-B |
| 07-D | `08-auth/auth-tests.md` | Auth unit + integration tests | 07-C |

### PHASE 8 — Observability & Hardening
| # | Skill File | What It Generates | Dependency |
|---|---|---|---|
| 08-A | `10-observability/otel-setup.md` | OpenTelemetry tracing setup | All phases |
| 08-B | `10-observability/structured-logging.md` | structlog config (Splunk-compatible) | 08-A |
| 08-C | `10-observability/docker-compose.md` | docker-compose for local dev | All phases |
| 08-D | `10-observability/dockerfile-per-service.md` | Dockerfile per MCP server | 05-A |

---

## Skill File Anatomy

Every skill file follows this structure:
```
# Skill: [Name]
## Context       — what this component does and why
## Parameters    — {{PLACEHOLDERS}} to fill in
## Dependencies  — what must already exist
## Prompt        — the actual text to paste into Copilot
## Expected Output — files/classes Copilot should produce
## Verification  — how to confirm it worked
```
