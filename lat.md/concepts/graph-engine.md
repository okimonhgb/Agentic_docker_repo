# Graph Engine

The graph engine dynamically constructs LangGraph `StateGraph` instances from JSON configuration files. This enables rapid iteration on agent workflows without code changes — graphs are defined as data, not hard-coded.

## Builder

The core of the graph engine is [[agentic/graph_engine/builder.py#build_graph]], which:
1. Reads a JSON config file specifying nodes, edges, and conditional routes.
2. Instantiates agent nodes via `AgentFactory`.
3. Wires nodes with edges (both static and conditional).
4. Compiles and returns a runnable LangGraph graph.

### JSON config format

Config files are stored in `agentic/graph_engine/configs/`. Each defines:
- `nodes` — List of agent nodes with names, types, and tool configurations.
- `edges` — Static edges between nodes.
- `conditional_edges` — Dynamic routing with a router function name.
- `entry_point` — The starting node.
- `tools_override` — Per-node tool configuration overrides.

### Router functions

Conditional routing uses named router functions. The builder resolves these from:
- Built-in routers: `supervisor_router`, `validator_router` (defined in builder.py).
- Optional routers: booking routers ([[agentic/agents/booking/routers.py]]), RAG routers ([[agentic/agents/rag/routers.py]]), supervisor routers ([[agentic/workflow_builder/agents/routers.py]]), and workflow builder routers ([[agentic/agents/workflow_builder/router.py]]).

## Agent factory

[[agentic/graph_engine/agent_factory.py#AgentFactory]] creates agent nodes from configuration. It:
- Resolves agent class by name from registered agents.
- Binds tools to agents based on config or defaults.
- Configures LLM model, temperature, and structured output schemas.
- Applies prompt templates with variable substitution.

## Graph execution flow

The main supervisor graph follows this execution pattern:

1. **Intent classification** — The `intent_classifier` node applies hard overrides (deterministic keyword matching for confirmations, single-word follow-ups) first, then falls back to LLM-based structured classification into `booking`, `rag`, or `general` intents. Defined in [[agentic/workflow_builder/agents/intent_classifier.py]].

2. **Routing** — `route_to_subgraph` (single intent) or `route_to_subgraphs` (multiple intents via LangGraph `Send` API for parallel execution). Defined in [[agentic/workflow_builder/agents/routers.py]].

3. **Subgraph execution** — The selected subgraph (booking, rag, or general_response) runs its internal workflow.

4. **Synthesis** — For multi-intent turns, the `synthesizer` node merges `partial_answers` from each subgraph into one coherent reply. Defined in [[agentic/workflow_builder/agents/synthesizer.py]].

5. **End** — The graph terminates and returns the final answer.

## State management

All agents share [[agentic/core/state.py#GraphState]], a TypedDict with custom reducers:
- `add_messages` — Appends to message list (LangGraph built-in).
- `merge_partial_answers` — Merges subgraph outputs for multi-intent synthesis; resets to empty on new turns.
- `take_last` — Last-value-wins for scalar fields, preventing `InvalidUpdateError` from parallel writes.
