# Skill: Project Scaffold
**Phase:** 0-A | **Type:** High-level Structure | **Depends on:** Nothing

---

## Context
Generates the complete folder structure, base config files, and `.gitignore` for the
multi-agent system monorepo. Run this first — every other skill depends on this layout.

---

## Parameters

| Parameter | Description |
|---|---|
| `{{PROJECT_NAME}}` | Root project folder name e.g. `mas_platform` |
| `{{PYTHON_VERSION}}` | e.g. `3.11` |
| `{{NODE_VERSION}}` | e.g. `18` |
| `{{NEXUS_REGISTRY_URL}}` | Internal Nexus base URL |

---

## Dependencies
- Git repo initialized
- Python `{{PYTHON_VERSION}}` available
- Node.js `{{NODE_VERSION}}` available

---

## Prompt

```
Generate the complete folder and file scaffold for a Python + TypeScript monorepo
called "{{PROJECT_NAME}}" with this exact structure:

{{PROJECT_NAME}}/
├── orchestrator/
│   ├── core/
│   │   ├── __init__.py
│   │   ├── llm_provider.py         # empty, filled by later skill
│   │   └── providers/
│   │       └── __init__.py
│   ├── agents/
│   │   └── __init__.py
│   ├── graph/
│   │   └── __init__.py
│   ├── tests/
│   │   └── __init__.py
│   ├── requirements.txt            # empty placeholder
│   └── pyproject.toml
│
├── skills-library/
│   ├── observability/
│   │   └── __init__.py
│   ├── workflow/
│   │   └── __init__.py
│   ├── knowledge/
│   │   └── __init__.py
│   ├── sdlc/
│   │   └── __init__.py
│   ├── base/
│   │   └── __init__.py
│   └── tests/
│       └── __init__.py
│
├── mcp-servers/
│   ├── appdynamics-mcp/
│   │   ├── src/
│   │   │   └── index.ts            # empty placeholder
│   │   ├── tests/
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── splunk-mcp/        (same structure as above)
│   ├── pagerduty-mcp/     (same structure as above)
│   ├── jira-mcp/          (same structure as above)
│   ├── confluence-mcp/    (same structure as above)
│   └── slack-mcp/         (same structure as above)
│
├── api-gateway/
│   ├── main.py                     # empty placeholder
│   ├── routers/
│   │   └── __init__.py
│   ├── auth/
│   │   └── __init__.py
│   ├── middleware/
│   │   └── __init__.py
│   └── tests/
│       └── __init__.py
│
├── docs/
│   └── architecture/
│
├── scripts/
│   └── verify_deps.py              # see below
│
├── .env.example
├── .gitignore
└── README.md

Rules:
- Python files use UTF-8 encoding headers
- All __init__.py files are empty
- package.json for each MCP server uses "private": true and points npm registry
  to "{{NEXUS_REGISTRY_URL}}"
- pyproject.toml uses Python {{PYTHON_VERSION}}
- .gitignore must exclude: .env, __pycache__, node_modules, dist, .venv, *.pyc,
  .pytest_cache, coverage/, *.egg-info

Also generate scripts/verify_deps.py — a script that reads requirements.txt and
package.json files across the monorepo and prints a checklist of all dependencies
with their versions, formatted as a table, so they can be submitted to the Nexus
admin for approval.

Output every file with full content. Do not skip any file.
```

---

## Expected Output
- Full directory tree created
- All placeholder `__init__.py` and stub files present
- `scripts/verify_deps.py` runnable with `python scripts/verify_deps.py`
- `.gitignore` excludes secrets and build artifacts

## Verification
```bash
# Run dependency audit immediately after scaffold
python scripts/verify_deps.py

# Check structure
find {{PROJECT_NAME}} -type f | head -60
```
