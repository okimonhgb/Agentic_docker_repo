# Agentic Service

Core LangGraph-based agentic workflow engine. A FastAPI application (port 8008) that builds, manages, and executes dynamic agent graphs. This is the "brain" of the platform — all LLM reasoning, tool use, and multi-agent orchestration happens here.

## Entry point

The application is started via [[agentic/app.py]] which loads environment variables, configures logging, and launches uvicorn on `api.main:app`.

## API layer

The FastAPI app is defined in [[agentic/api/main.py]]. Key endpoints:

### Chat execution

Endpoints for running agentic workflows and getting LLM-generated responses.

- `POST /run/with-graph` — Primary endpoint. Accepts a query with session context, runs the configured LangGraph workflow, and returns the final answer. Used by newui_test for every chat turn.
- `POST /run/{graph_id}` — Legacy endpoint for running a graph by its stored ID.

### Workflow management

CRUD endpoints for LangGraph workflow JSON configurations.

- `POST /api/workflow/save-with-id` — Save or update a workflow graph configuration (JSON). Called by platformui during app creation to register the initial graph.
- `GET /api/workflows` — List available workflows.
- Routes defined in `agentic/api/routes/workflows.py`.

### Booking webhooks

Webhook endpoints for receiving booking-related callbacks from external systems.

- `POST /api/webhooks/booking` — Receive booking-related callbacks (e.g., appointment created/cancelled).
- Defined in `agentic/api/webhooks/`.

### Session management

Endpoints for retrieving and managing conversation session state.

- `GET /api/sessions/{session_id}` — Retrieve session metadata and conversation history.
- Routes defined in `agentic/api/routes/sessions.py`.

### System endpoints

Operational endpoints for health checks, logging, and diagnostics.

- `GET /api/logs/stream` — SSE stream of server logs for debugging UI.
- `GET /health` — Health check.
- Routes defined in `agentic/api/routes/system.py`.

## Graph engine

The graph engine dynamically constructs LangGraph `StateGraph` instances from JSON configuration files. See [[concepts/graph-engine]] for details.

Key components:
- [[agentic/graph_engine/builder.py]] — Reads JSON configs, instantiates agents via `AgentFactory`, wires nodes and conditional edges, and returns a compiled graph.
- [[agentic/graph_engine/agent_factory.py]] — Creates agent nodes from configuration, resolving tool bindings and prompt templates.

## Core infrastructure

Shared foundational modules for state management, configuration, database access, and routing models.

- [[agentic/core/state.py#GraphState]] — Shared TypedDict state with reducers for message accumulation, partial answer merging, and last-value-wins semantics.
- [[agentic/core/config.py#Settings]] — Centralized, frozen dataclass settings loaded from environment variables.
- [[agentic/core/database.py]] — Database session management, delegates to `core.db.connection`.
- [[agentic/core/db/connection.py]] — SQLAlchemy engine and session factory for PostgreSQL.
- [[agentic/core/models/routing.py#SupervisorOutput]] — Pydantic model for structured LLM routing decisions.

## Subgraphs

The main supervisor graph delegates to specialized subgraphs:

### Booking subgraph
Handles appointment scheduling, availability checks, rescheduling, cancellations, and owner administrative edits. See [[concepts/booking]].
- Agents in `agentic/agents/booking/` — intent_agent, datetime_agent, availability_agent, negotiation_agent, booking_agent, list_appointments_agent, list_experts_agent, admin_availability_agent, owner_edit_agent, response_agent.
- Domain logic in `agentic/booking/` — booking_manager, slot_state_machine, context_codec, organization_manager, owner_edit_planner.

### RAG subgraph
Handles document-grounded question answering. See [[concepts/rag]].
- Agents in `agentic/agents/rag/` — intake_agent, router_agent, chat_agent, clarify_agent, rewrite_query_agent, retrieve_agent, answer_rag_agent, grade_documents_agent.

### Workflow builder subgraph
Handles natural-language workflow creation. See [[concepts/workflow-builder]].
- Agents in `agentic/agents/workflow_builder/` — intent_analyzer, capabilities_agent, clarification_agent, feasibility_checker, graph_generator, proposal_summarizer, workflow_coordinator.

## External messaging

The `agentic/external_messaging/` module provides an abstraction for outbound messaging (WhatsApp, SMS). Uses a provider registry pattern:
- [[agentic/external_messaging/registry.py]] — Maps provider names to implementations.
- [[agentic/external_messaging/providers/twilio_whatsapp.py]] — Twilio WhatsApp provider.

## Toolbox

Reusable agents, tools, retrievers, and prompt templates. The "dumb" layer that knows nothing about graphs:
- `agentic/toolbox/agents/` — Generic agent implementations (supervisor, researcher).
- `agentic/toolbox/tools/` — Reusable tools (web search, Wikipedia, RAG tool).
- `agentic/toolbox/retrievers/` — ChromaDB manager, document loader, retriever factory.
- `agentic/toolbox/prompts/` — Prompt template loader with multilingual support.
- `agentic/toolbox/utils/` — Language policy, phrase matcher utilities.

## Configuration files

JSON and INI config files that drive multilingual behavior, LLM settings, and domain taxonomies.

- `agentic/config/booking_keywords.json` — Multilingual booking intent keywords (Turkish, English).
- `agentic/config/llm_models.ini` — LLM model configurations per provider.
- `agentic/config/sector_taxonomy.json` — Business sector taxonomy for onboarding.
- `agentic/config/address_keywords.json` — Address-related keyword patterns.
