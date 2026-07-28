# Memory System

Multi-tier conversation memory providing persistent chat history, automatic context summarization, agent execution traces, and session state management. Uses MongoDB as the storage backend.

## Smart memory handler

The [[agentic/memory/smart_memory_handler.py]] module is the primary interface for all memory operations:

### Conversation storage

Persists and retrieves chat messages for building LLM context windows.

- `save_conversation()` — Persists a user/bot message pair to the `chat_sessions` MongoDB collection.
- `get_conversation_history()` — Retrieves recent messages for context window construction, with automatic summarization of older messages.

### Agent traces

Records agent execution details for debugging and performance analytics.

- `save_agent_trace()` — Records agent execution details (node name, input/output, latency, tokens) for debugging and analytics. Stored in `agent_traces` collection.

### Context summarization

Automatically summarizes older messages to keep context windows within token limits.

- Automatic summarization triggers when message count exceeds `MAX_MESSAGES_BEFORE_SUMMARIZATION` (default 20).
- Older messages are summarized via LLM and stored in `context_summaries`, while the most recent `TARGET_CONTEXT_MESSAGES` (default 10) are kept in full.
- Summaries are injected as `SystemMessage` at the start of each turn's context.

### Session state

Retrieves and manages booking/RAG state snapshots for conversation continuity.

- `get_last_state_snapshot()` — Retrieves the most recent booking/RAG state snapshot for stateful conversation continuity.
- `get_session_state_fields()` — Reads specific state fields (e.g., `pending_booking_confirmation`) for gating logic.
- `clear_session()` — Resets session state for a fresh conversation.

### Booking keyword cache

Caches booking keywords from config for fast intent detection without disk I/O.

- Caches the contents of `agentic/config/booking_keywords.json` for intent detection, with file mtime-based invalidation.

## Customer memory store

[[agentic/memory/customer_memory_store.py]] maintains per-customer profiles including:
- Contact information (name, phone, email).
- Appointment history.
- Preferences and notes.

## Business knowledge store

[[agentic/memory/business_knowledge_store.py]] stores organization-specific knowledge:
- Business hours, policies, and FAQs.
- Service descriptions and pricing.
- Resource/Staff profiles.

## MongoDB collections

MongoDB collections used for conversation storage and operational data.

| Collection | Purpose |
|---|---|
| `chat_sessions` | Conversation messages per session |
| `agent_traces` | Agent execution logs for debugging |
| `context_summaries` | LLM-generated summaries of old messages |
| `archived_messages` | Messages removed from active context |

## Integration with LangGraph

Memory is read before graph invocation and written after — not part of LangGraph state.

Memory is not part of the LangGraph state directly. Instead, the API layer in [[agentic/api/main.py]] reads conversation history from MongoDB before each graph invocation, constructs the `messages` list, and passes it as part of the initial state. After execution, the new messages are persisted back to MongoDB.
