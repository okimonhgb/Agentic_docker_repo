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
