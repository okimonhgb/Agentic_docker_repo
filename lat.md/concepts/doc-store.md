# Document Store (Dual Backend)

Unified abstraction over MongoDB and PostgreSQL for schema-less document storage.

The document store provides a unified interface for chat sessions, traces, and message stores. It mirrors the vector store dual-backend pattern, with `DOC_STORE_BACKEND` env var selecting between MongoDB and PostgreSQL JSONB storage.

## Architecture

`DocStoreManager` (async) and `SyncDocStoreManager` (sync) delegate to backend-specific adapters. When `postgresql` is selected, documents are stored as JSONB rows in `doc_<collection>` tables with GIN indexes.

## Backends

Two storage backends are supported, selected at runtime.

- **MongoDB** — Uses Motor (async) or pymongo (sync). Collections are created on-demand.
- **PostgreSQL** — Uses SQLAlchemy with JSONB columns. Tables are auto-created via `ensure_table()`. GIN indexes on the `data` column enable efficient JSONB key lookups.

## Collections per service

Each service has its own set of document collections mapped to PostgreSQL tables.

### agentic/

Async via `DocStoreManager`. Collections: `chat_sessions`, `agent_traces`, `context_summaries`, `archived_messages`.

| Collection | Table | Purpose |
|---|---|---|
| `chat_sessions` | `doc_chat_sessions` | Session metadata, state fields, messages |
| `agent_traces` | `doc_agent_traces` | Agent execution traces with turns |
| `context_summaries` | `doc_context_summaries` | LLM-generated conversation summaries |
| `archived_messages` | `doc_archived_messages` | Archived messages before summarization |

### newui_test/

Sync via `SyncDocStoreManager`. Collections: `llm_message_store`, `session_ai_state`.

| Collection | Table | Purpose |
|---|---|---|
| `llm_message_store` | `doc_llm_message_store` | Chat message persistence |
| `session_ai_state` | `doc_session_ai_state` | AI pause/resume state |

### etl/

Sync via `ETLDocStoreManager`. Collection: `doc_summary`.

| Collection | Table | Purpose |
|---|---|---|
| `doc_summary` | `doc_doc_summary` | Document summaries with bigchat_id index |

## Filter translation

The PostgreSQL adapter translates MongoDB-style filters to SQL JSONB queries.

- `{field: value}` → `data @> CAST(:param AS jsonb)` (JSONB containment)
- `{field: {"$ne": value}}` → `NOT (data @> CAST(:param AS jsonb))`
- `{field: {"$in": [...]}}` → `data->>'field' IN (...)`
- `{field: {"$gte": N}}` → `(data->>'field')::text >= :param`
- `{field: {"$exists": true}}` → `data ? 'field'`

## Key files

Core abstraction and consumer files across services.

- `agentic/core/db/doc_store.py` — Async abstraction for agentic/
- `newui_test/core/doc_store.py` — Sync abstraction for newui_test/
- `etl/libs/doc_store.py` — Sync abstraction for etl/
- `agentic/memory/smart_memory_handler.py` — Consumer in agentic/
- `agentic/rag/rag_manager.py` — Consumer in RAG reporting

## Migration

See [[decisions#MongoDB to PostgreSQL Migration]] for the decision record and `docs/MONGODB_TO_POSTGRES_MIGRATION_PLAN.md` for the full plan.

## Verification tests

Test suites verify CRUD operations, integration, data integrity, and backend configuration after migration.

- `agentic/tests/test_migration_verify.py` — 26 tests: DocStoreManager CRUD, smart_memory_handler integration, MongoDB vs PostgreSQL count integrity, JSONB queryability, GIN index existence, backend config
- `agentic/tests/test_doc_store_unit.py` — 17 tests: DocStoreConfig backend selection, _PostgresDocStore filter translation, _MongoDocStore delegation (mocked, no DB required)
- `newui_test/tests/test_doc_store_migration.py` — 15 tests: SyncDocStoreManager CRUD, message_service integration, data integrity, GIN indexes, backend config
