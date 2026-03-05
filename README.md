# Multi-Agent-LangGraph
# Multi-Agent System with Skills Library & MCP Integration

> **Status:** Planning / Architecture Phase  
> **Last Updated:** 2026-03  
> **Owner:** AI Strategy & Implementation Team

---

## Overview

This repository documents and implements a **loosely coupled, model-agnostic, multi-agent system** designed to operate within organizational constraints. The system routes user tasks to specialized agents that invoke skills backed by MCP (Model Context Protocol) servers — each independently deployable and testable.

### Primary Use Cases (Priority Order)
1. **DevOps & Observability** — Automated alert triage, incident creation, runbook lookup
2. **Cross-team Workflow Automation** — Task routing, approvals, escalations
3. **Knowledge Retrieval** — Confluence search, document summarization
4. **SDLC Automation** — PR review, deploy status, code context

---

## Repository Structure

```
multi-agent-system/
│
├── README.md                          ← You are here
│
├── docs/
│   ├── architecture/
│   │   ├── ARCHITECTURE.md            ← Full system design & component breakdown
│   │   ├── COMPONENT_CONTRACTS.md     ← Interface contracts between components
│   │   └── HYBRID_DEPLOYMENT.md       ← On-prem vs cloud boundary design
│   │
│   ├── planning/
│   │   ├── PHASED_PLAN.md             ← Phased delivery roadmap
│   │   ├── DEPENDENCIES.md            ← All dependencies for Nexus validation
│   │   └── CONSTRAINTS.md             ← Org constraints & design decisions
│   │
│   └── guides/
│       ├── MCP_SERVER_GUIDE.md        ← How to build & publish an MCP server
│       ├── SKILLS_GUIDE.md            ← How to write and register a skill
│       ├── TESTING_GUIDE.md           ← Independent testing per component
│       └── AGENT_PROVIDER_GUIDE.md    ← How to plug into org Agent Provider
│
├── src/
│   ├── orchestrator/                  ← LangGraph orchestration engine
│   ├── agents/                        ← Specialist agent definitions
│   ├── skills/                        ← Skills library (per domain)
│   ├── mcp-servers/                   ← Individual MCP server implementations
│   ├── llm-adapter/                   ← Model-agnostic LLM interface
│   └── shared/                        ← Shared contracts, types, utilities
│
└── tests/
    ├── unit/                          ← Per-component unit tests
    ├── integration/                   ← Cross-component integration tests
    └── mocks/                         ← Mock MCP servers & LLM stubs
```

---

## Quick Navigation

| I want to...                              | Go to |
|-------------------------------------------|-------|
| Understand the full system design         | [ARCHITECTURE.md](docs/architecture/ARCHITECTURE.md) |
| See what phases we build in               | [PHASED_PLAN.md](docs/planning/PHASED_PLAN.md) |
| Check dependency list for Nexus           | [DEPENDENCIES.md](docs/planning/DEPENDENCIES.md) |
| Understand org constraints we design for  | [CONSTRAINTS.md](docs/planning/CONSTRAINTS.md) |
| Build a new MCP server                    | [MCP_SERVER_GUIDE.md](docs/guides/MCP_SERVER_GUIDE.md) |
| Write a new skill                         | [SKILLS_GUIDE.md](docs/guides/SKILLS_GUIDE.md) |
| Test a component independently            | [TESTING_GUIDE.md](docs/guides/TESTING_GUIDE.md) |
| Plug into org Agent Provider later        | [AGENT_PROVIDER_GUIDE.md](docs/guides/AGENT_PROVIDER_GUIDE.md) |

---

## Design Principles

These principles are non-negotiable and drive every architectural decision:

| Principle | What it means |
|-----------|---------------|
| **Model Agnostic** | LLM is swappable via adapter — no vendor lock-in |
| **Loosely Coupled** | Every component has a defined contract; none depend on internals of another |
| **Independently Testable** | Every component can be tested in isolation with mocks |
| **Org Constraint First** | Works within locked-down networks, Nexus repos, no external pypi |
| **MCP First** | All tool integrations are MCP servers — publishable to org platform |
| **Future Agent Provider Ready** | Agents and MCP servers can be handed off to org's common agent platform |

---

## Getting Started

> Detailed setup instructions will be added as implementation begins in Phase 1.

**Prerequisites checklist (validate against Nexus before starting):**
- [ ] Python 3.11+
- [ ] Node.js 20+
- [ ] All packages from [DEPENDENCIES.md](docs/planning/DEPENDENCIES.md) available in Nexus
- [ ] Internal network access to target MCP tool endpoints
- [ ] LLM endpoint URL and auth token from org Model Directory

---

## Contributing

See [SKILLS_GUIDE.md](docs/guides/SKILLS_GUIDE.md) for adding new skills and [MCP_SERVER_GUIDE.md](docs/guides/MCP_SERVER_GUIDE.md) for adding new MCP servers. All contributions must include unit tests per [TESTING_GUIDE.md](docs/guides/TESTING_GUIDE.md).
