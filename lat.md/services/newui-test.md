# newui_test Service

Chat API frontend for the platform. A FastAPI application (port 8000) that receives user messages, resolves org config, routes to agentic, and persists conversation history.

## Entry point

The application is started via [[newui_test/app.py]] which launches uvicorn on `api.main:app`.

## API layer

HTTP and WebSocket endpoints for chat interactions and conversation history.

### Chat endpoints

Primary HTTP endpoints for sending messages and retrieving history.

- `POST /api/chat/{chatbot_id}` — Main chat endpoint. Saves the user message to MongoDB, calls `llm_service.get_llm_response()` to forward to agentic, saves the bot response, and returns it. Also optionally pushes via WebSocket.
- `GET /api/session/{session_id}/history` — Retrieve persisted messages for a session. Called by the customer chat widget on page load so the conversation survives a browser refresh. Enforces that the authenticated user matches the session's embedded user id.

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

Static assets and Jinja2 templates for both the admin test UI and the public customer chat UI.

- `static/css/stiltai-ui.css` — Shared styles, including the customer chat (`cc-*`) classes.
- `templates/customer_chat_shell.html` — Shell for the public `/b/{slug}` chat page. Loads the customer chat partial and hosts `window.initCustomerChat`, which manages the session id, WebSocket, message rendering, history restore, and now a **"New chat" reset**.
- `templates/partials/customer_chat.html` — Public chat UI rendered inside the shell. Includes the brand header, language switch, logout button, and the new **"New chat"** button.
- `templates/partials/test_conversation.html` — Admin test chat UI used for internal testing. It has its own sidebar, session history list, and a "New chat" button.

### Session handling in customer chat

The public chat uses a deterministic base session id so a returning customer resumes the same MongoDB-backed conversation across devices.

The **"New chat"** button lets users escape a stale conversation state without logging out. It generates a fresh timestamped session id, stores it in `localStorage`, clears the visible history, and reconnects the WebSocket.

### Duplicate message prevention

The bot reply is delivered via both the HTTP response and a WebSocket push, and HTMX re-swaps can create multiple script instances — both cause duplicate messages.

The chat route (`POST /api/chat/{chatbot_id}`) saves the bot message, pushes it via `ConnectionManager.send_message`, and returns it in the HTTP response. The client template `test_conversation.html` must avoid rendering the same bot message twice when both channels carry it.

HTMX partial re-swaps can re-insert and re-execute the inline `<script>` tag, creating multiple independent script instances. Each instance opens its own WebSocket and maintains its own dedup state (`seenMessageIds`, `httpBotAddedThisTurn`), so two instances can each render the same bot answer once — one via HTTP, the other via its WebSocket push — producing a visible duplicate.

A window-level instance counter (`window.__tcInstanceCounter`) implements a "latest instance wins" guard. Each execution increments the counter and records its own `myInstanceId`. The `isStaleInstance()` helper returns `true` when a newer instance has since been created. All entry points (`sendMessage`, `connectWs`, `ws.onopen`, `ws.onmessage`) bail out immediately if the instance is stale, so only the most recent script instance actively processes messages.
