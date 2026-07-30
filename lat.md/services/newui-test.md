# newui_test Service

Chat API frontend for the platform. A FastAPI application (port 8000) that receives user messages, resolves org config, routes to agentic, and persists conversation history.

## Entry point

The application is started via [[newui_test/app.py]] which launches uvicorn on `api.main:app`.

## API layer

HTTP and WebSocket endpoints for chat interactions and conversation history.

### Chat endpoints

Primary HTTP endpoints for sending messages and retrieving history.

- `POST /api/chat/{chatbot_id}` — Main chat endpoint. Saves the user message to MongoDB, calls `llm_service.get_llm_response()` to forward to agentic, saves the bot response, and returns it. Also optionally pushes via WebSocket.
- `GET /api/chat/{chatbot_id}/history` — Retrieve conversation history for a session.

### WebSocket

Real-time push channel for live message delivery to connected clients.

- `WS /ws/{chatbot_id}/{session_id}` — Real-time message push channel. Best-effort delivery; the HTTP response is the primary path.

## Services layer

Business logic for LLM invocation, message persistence, and chatbot creation.

- `services/llm_service.py` — Resolves chatbot configuration (org_id, graph_id) from PostgreSQL, builds the LangGraph payload, and calls agentic's `/run/with-graph` endpoint.
- `services/message_service.py` — MongoDB operations for saving/retrieving user and bot messages.
- `services/chatbot_creation_service.py` — Orchestrates new chatbot/app creation, including ChromaDB collection setup and booking organization bootstrap.

## Core infrastructure

Shared modules for configuration, database connections, and MongoDB access.

- `core/config.py` — Application configuration from environment variables.
- `core/database.py` — PostgreSQL connection for reading chatbot/organization metadata.
- `core/mongodb.py` — MongoDB connection for conversation storage.

## Frontend

Static assets and templates for the chat UI and any server-rendered pages.

- `static/` — Contains the chat UI (`test_conversation.html`) used for testing and demonstration.
- `templates/` — Jinja2 templates if any server-rendered pages exist.

## Duplicate message prevention

The bot reply is delivered via both the HTTP response and a WebSocket push, and HTMX re-swaps can create multiple script instances — both cause duplicate messages.

The chat route (`POST /api/chat/{chatbot_id}`) saves the bot message, pushes it via `ConnectionManager.send_message`, and returns it in the HTTP response. The client template `test_conversation.html` must avoid rendering the same bot message twice when both channels carry it.

HTMX partial re-swaps can re-insert and re-execute the inline `<script>` tag, creating multiple independent script instances. Each instance opens its own WebSocket and maintains its own dedup state (`seenMessageIds`, `httpBotAddedThisTurn`), so two instances can each render the same bot answer once — one via HTTP, the other via its WebSocket push — producing a visible duplicate.

A window-level instance counter (`window.__tcInstanceCounter`) implements a "latest instance wins" guard. Each execution increments the counter and records its own `myInstanceId`. The `isStaleInstance()` helper returns `true` when a newer instance has since been created. All entry points (`sendMessage`, `connectWs`, `ws.onopen`, `ws.onmessage`) bail out immediately if the instance is stale, so only the most recent script instance actively processes messages.
