# Supervisor and Intent Routing

The supervisor is the top-level orchestrator that classifies every user message and routes it to the appropriate subgraph(s). It uses a hybrid approach: fast deterministic rules for common patterns, with LLM-based classification as fallback.

## Intent classifier

The entry point for every chat turn. Defined in [[agentic/workflow_builder/agents/intent_classifier.py]].

### Hard overrides (deterministic, no LLM call)

Applied in order before any LLM classification:
1. **Confirmation in booking context** — Bare "yes/no/evet/hayır" when `pending_booking_confirmation` or `pending_cancel_confirmation` is set → routes to booking subgraph directly.
2. **Pending owner edit** — When an owner edit is awaiting confirmation → routes to booking subgraph.
3. **Single-word follow-up** — A single word matching a known service/resource name → routes to booking subgraph.

### LLM classification

If no hard override matches, the classifier calls an LLM with structured output to produce an `IntentOutput` containing one or more of: `booking`, `rag`, `general`.

## Multi-intent routing

[[agentic/workflow_builder/agents/routers.py]] provides:
- `route_to_subgraph` — Routes a single intent to its matching subgraph.
- `route_to_subgraphs` — For multiple intents, uses LangGraph's `Send` API to execute subgraphs in parallel, then routes to the synthesizer.
- `route_after_subgraph` — After subgraph completion, routes to synthesizer (if more subgraphs pending) or END.

## Synthesizer

[[agentic/workflow_builder/agents/synthesizer.py]] merges `partial_answers` from parallel subgraph executions into one coherent, non-redundant reply. It handles deduplication of overlapping information (e.g., if both RAG and booking mention the same service) and natural language merging.

## General response

[[agentic/workflow_builder/agents/general_response.py]] handles queries outside booking and RAG scope: chit-chat, identity questions ("who am I?"), and general knowledge. Uses a lightweight LLM call without tool binding.

## Supervisor node factory

[[agentic/toolbox/agents/supervisor.py#create_supervisor_node]] is a generic, reusable node factory for creating supervisor-style routing nodes. It takes a team list, output model, and optional prompt override. Used by the graph builder to construct supervisor nodes dynamically from JSON config.
