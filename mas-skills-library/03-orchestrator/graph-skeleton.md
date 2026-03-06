# Skill: Orchestrator — LangGraph Graph Skeleton
**Phase:** 2-A | **Type:** High-level Structure | **Depends on:** 02-E (state-schema), 01-D (factory)

---

## Context
The LangGraph state machine that wires all nodes together. Defines the execution
flow: classify intent → decompose task → route to agent → collect result → respond.
Uses conditional edges so routing is dynamic at runtime.

---

## Parameters

| Parameter | Description |
|---|---|
| `{{PROJECT_NAME}}` | Module docstring |

---

## Dependencies
- `02-E` OrchestratorState defined
- `01-D` get_llm_provider() available
- `requirements.txt` has `langgraph`, `langchain-core`

---

## Prompt

```
Generate the file: orchestrator/graph/graph.py

Build the LangGraph StateGraph for the multi-agent orchestrator.
Use OrchestratorState as the state type. Python 3.11+.

Structure to generate:

1. Node functions (async, each takes state: OrchestratorState, returns dict of updates)

   a. classify_intent_node(state, llm) -> dict
      - Builds a prompt: given raw_input, classify intent into one of:
        [alert_triage, incident_management, runbook_lookup, code_review,
         deploy_check, cross_team_routing, unknown]
      - Calls llm.complete() with a system prompt + user message
      - Parses the response to extract intent string + confidence float
      - Returns {"intent": ..., "intent_confidence": ..., "status": ROUTING}
      - On LLMProviderError: returns {"intent": "unknown", "intent_confidence": 0.0}

   b. decompose_task_node(state, llm) -> dict
      - Only runs if intent_confidence > 0.6
      - Prompts LLM to break the task into max 3 ordered steps
      - Returns {"decomposed_steps": [...]}
      - On failure: returns {"decomposed_steps": [state.task.raw_input]}

   c. route_to_agent_node(state) -> dict
      DETERMINISTIC (no LLM call). Rules:
      - "alert_triage" or "incident_management" → AgentType.DEVOPS
      - "runbook_lookup" → AgentType.KNOWLEDGE
      - "code_review" or "deploy_check" → AgentType.SDLC
      - "cross_team_routing" → AgentType.WORKFLOW
      - anything else → AgentType.DEVOPS (safe default)
      Returns {"assigned_agent": ...}

   d. agent_dispatch_node(state, agent_registry) -> dict
      - Looks up the assigned_agent in agent_registry (dict: AgentType → agent instance)
      - Calls agent.run(state) → AgentResult
      - Returns {"agent_results": [result], "status": COMPLETED or FAILED}

   e. format_response_node(state) -> dict
      - Compiles final_response from agent_results
      - Returns {"final_response": ..., "status": COMPLETED}

2. build_graph(llm_provider, agent_registry) -> CompiledGraph
   - Create StateGraph(OrchestratorState)
   - Add all 5 nodes
   - Add edges:
     START → classify_intent_node
     classify_intent_node → decompose_task_node
     decompose_task_node → route_to_agent_node
     route_to_agent_node → agent_dispatch_node
     agent_dispatch_node → format_response_node
     format_response_node → END
   - Add conditional edge from classify_intent_node:
     if status == FAILED → END (skip remaining nodes)
   - Return graph.compile()

3. async run_task(raw_input, source, llm_provider, agent_registry,
                  priority=TaskPriority.MEDIUM, metadata={}) -> OrchestratorState
   Convenience wrapper:
   - Builds IncomingTask
   - Builds initial OrchestratorState
   - Calls graph.ainvoke(state)
   - Returns final state
   - Logs task_id, duration, final status using structlog

Rules:
- Node functions use functools.partial to inject llm/agent_registry dependencies
- structlog for all logging
- state.to_summary_dict() for log entries (never log raw_input)
- Full type annotations
```

---

## Expected Output
- `orchestrator/graph/graph.py` ~150 lines
- `build_graph()` callable
- `run_task()` is the single public entry point

## Verification
```python
import asyncio, os
os.environ["LLM_PROVIDER"] = "mock"
from orchestrator.graph.graph import run_task
from orchestrator.core.providers.factory import get_llm_provider

async def test():
    result = await run_task("test alert", "api", get_llm_provider(), {})
    print(result.status)

asyncio.run(test())
```
