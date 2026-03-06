# Skill: Nexus Registry Configuration
**Phase:** 0-B | **Type:** High-level Config | **Depends on:** 00-A (project-scaffold)

---

## Context
Configures all package managers (pip, npm) to resolve dependencies exclusively from
your organization's Nexus repository. This ensures no packages are pulled from the
public internet, satisfying org security constraints.

---

## Parameters

| Parameter | Description |
|---|---|
| `{{NEXUS_REGISTRY_URL}}` | Full Nexus base URL e.g. `https://nexus.internal.org` |
| `{{NEXUS_PYPI_REPO}}` | Nexus PyPI proxy repo path e.g. `/repository/pypi-proxy/simple/` |
| `{{NEXUS_NPM_REPO}}` | Nexus npm proxy repo path e.g. `/repository/npm-proxy/` |
| `{{PROJECT_NAME}}` | Root project name |

---

## Dependencies
- `00-A` scaffold complete

---

## Prompt

```
Generate the following Nexus-locked package manager configuration files for the
monorepo "{{PROJECT_NAME}}". All packages must resolve ONLY from the internal
Nexus registry — never from public registries.

1. pip.conf (place at project root and also at orchestrator/ and api-gateway/)
Content should:
- Set index-url to {{NEXUS_REGISTRY_URL}}{{NEXUS_PYPI_REPO}}
- Set trusted-host to the Nexus hostname
- Disable fallback to pypi.org

2. .npmrc (place at project root and inside each mcp-servers/* folder)
Content should:
- Set registry=  to {{NEXUS_REGISTRY_URL}}{{NEXUS_NPM_REPO}}
- Set strict-ssl=true
- Set always-auth=false (auth handled separately)

3. orchestrator/requirements.txt
Full pinned dependency list:
langgraph==0.2.28
langchain-core==0.3.15
langchain==0.3.7
openai==1.51.0
httpx==0.27.2
mcp==1.1.0
fastapi==0.115.4
uvicorn==0.30.6
pydantic==2.9.2
pydantic-settings==2.5.2
redis==5.1.1
python-dotenv==1.0.1
structlog==24.4.0
tenacity==8.5.0
opentelemetry-sdk==1.27.0
opentelemetry-exporter-otlp==1.27.0
pytest==8.3.3
pytest-asyncio==0.24.0
pytest-mock==3.14.0

4. api-gateway/requirements.txt
Same as orchestrator/requirements.txt plus:
python-ldap==3.4.4
python-jose==3.3.0
passlib==1.7.4

5. mcp-servers/package-base.json
A base package.json template (to be copied into each mcp-servers/* folder) with:
- devDependencies: typescript@5.6.3, ts-node@10.9.2, jest@29.7.0, ts-jest@29.2.5,
  @types/jest@29.5.13, esbuild@0.24.0
- dependencies: @modelcontextprotocol/sdk@1.0.0, axios@1.7.7, zod@3.23.8,
  dotenv@16.4.5, winston@3.15.0
- scripts section: build, test, start, dev
- All registry references pointing to {{NEXUS_REGISTRY_URL}}{{NEXUS_NPM_REPO}}

6. mcp-servers/tsconfig-base.json
Base TypeScript config with:
- target: ES2022
- module: NodeNext
- moduleResolution: NodeNext
- strict: true
- outDir: ./dist
- rootDir: ./src
- esModuleInterop: true

Each file should be complete and production-ready. Add inline comments explaining
each config decision.
```

---

## Expected Output
- `pip.conf` in project root, `orchestrator/`, `api-gateway/`
- `.npmrc` in project root and each `mcp-servers/*/`
- `orchestrator/requirements.txt` with all pinned versions
- `api-gateway/requirements.txt` with auth extras
- `mcp-servers/package-base.json` template
- `mcp-servers/tsconfig-base.json`

## Verification
```bash
# Verify pip resolves from Nexus only
pip install -r orchestrator/requirements.txt --dry-run

# Verify npm resolves from Nexus only
cd mcp-servers/appdynamics-mcp && npm install --dry-run
```
