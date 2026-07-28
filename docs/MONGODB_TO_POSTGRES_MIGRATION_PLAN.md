# MongoDB → PostgreSQL Migration Plan

## Goal

Migrate all MongoDB usage to PostgreSQL, supporting **both MongoDB and PostgreSQL** as runtime-selectable backends via an env var (`DOC_STORE_BACKEND=mongodb|postgresql`), following the same dual-backend pattern already used for ChromaDB/pgvector in `agentic/core/db/vector_store.py`.

## Scope

Only `channel/`, `etl/`, `newui_test/`, and `agentic/` repos are in scope.

- **`channel/`** — Already SQLAlchemy-only (PostgreSQL/MySQL). No MongoDB usage. **No changes needed.**
- **`newui_test/`** — MongoDB via `pymongo` (sync) for message storage and session AI state.
- **`agentic/`** — MongoDB via `motor` (async) for chat sessions, agent traces, context summaries, archived messages. Also used by `rag_manager.py`.
- **`etl/`** — MongoDB via `pymongo` (sync) in `HelixBase` and `MongoDBDocSummary`.

---

## Current MongoDB Usage (Inventory)

### `agentic/` — 4 collections in `agentic_project` DB

| Collection | File | Purpose |
|---|---|---|
| `chat_sessions` | `memory/smart_memory_handler.py` | Session metadata, state fields, conversation messages |
| `agent_traces` | `memory/smart_memory_handler.py` | Agent execution traces (debugging/analytics) |
| `context_summaries` | `memory/smart_memory_handler.py` | Conversation summaries |
| `archived_messages` | `memory/smart_memory_handler.py` | Archived messages |

**Key functions:** `get_mongo_client()`, `save_conversation()`, `save_agent_trace()`, `get_conversation_history()`, `get_last_state_snapshot()`, `get_session_state_fields()`, `clear_session()`, `get_session_info()`, `_summarize_conversation()`, `close_mongo_connection()`

**Also used by:** `rag/rag_manager.py` (`_get_mongo_client()` reads `agent_traces` and `chat_sessions`)

### `newui_test/` — 2 collections in `Buyingen` DB

| Collection | File | Purpose |
|---|---|---|
| `llm_message_store` (configurable) | `services/message_service.py` | Chat message persistence |
| `session_ai_state` | `services/session_ai_service.py` | Session AI pause/resume state |

**Key functions:** `save_message()`, `get_session_history()`, `get_all_sessions()` (uses aggregation pipeline), `pause_session()`, `resume_session()`, `is_session_paused()`

**Config:** `core/config.py` (`MONGODB_CONNECTION_STRING`, `MONGODB_DATABASE`, `MONGODB_MESSAGE_COLLECTION`), `core/database.py` (`init_database()`, `get_mongodb_collection()`, `get_mongodb_db()`)

### `etl/` — MongoDB in 2 files

| File | Purpose |
|---|---|
| `libs/base.py` (`HelixBase`) | `self.mongo = pymongo.MongoClient(...)` — general-purpose MongoDB handle |
| `my_works/connectors/MongoDBDocSummary.py` | Document summary storage (per `bigchat_id`) |

### `channel/` — No MongoDB usage

Already uses SQLAlchemy (PostgreSQL/MySQL) via `app/db/session.py`. No changes needed.

---

## Existing Dual-Backend Pattern (Reference)

The codebase already has a proven dual-backend pattern for vector stores:

- `agentic/core/db/vector_store.py` — `VectorStoreManager` + `VectorStoreConfig`, selects backend via `VECTOR_STORE_BACKEND` env var
- `etl/libs/vector_dbs/__init__.py` — Same pattern for ETL
- Both have adapter classes (`_ChromaManagerAdapter`, `_PGVectorManager`) implementing the same interface

We will follow this exact pattern for document stores.

---

## Step-by-Step Plan

### Phase 1: Design the Abstraction Layer

#### Step 1.1 — Create `agentic/core/db/doc_store.py`

New file defining the unified document store interface:

```
DocStoreConfig (dataclass)
  - backend: str (env: DOC_STORE_BACKEND, default: "mongodb")
  - pg_connection: Optional[str] (reuses POSTGRES_* env vars)
  - mongo_uri: str (env: MONGO_URI or MONGODB_CONNECTION_STRING)
  - mongo_db_name: str (env: MONGO_DB or "agentic_project")

DocStoreManager
  - __init__(config) → selects _MongoDocStore or _PostgresDocStore
  - async get_collection(name) → returns a DocCollection abstraction
  - async insert_one(collection, doc) → str (id)
  - async insert_many(collection, docs) → List[str]
  - async find_one(collection, filter) → Optional[dict]
  - async find_many(collection, filter, sort, limit) → List[dict]
  - async update_one(collection, filter, update) → bool
  - async replace_one(collection, filter, doc, upsert) → bool
  - async delete_one(collection, filter) → bool
  - async delete_many(collection, filter) → int
  - async aggregate(collection, pipeline) → List[dict]  # translate common pipelines
  - async count_documents(collection, filter) → int
  - async create_index(collection, field, unique) → None
  - close()
```

**Adapter classes:**
- `_MongoDocStore` — wraps `motor.motor_asyncio.AsyncIOMotorClient` (existing behavior)
- `_PostgresDocStore` — uses SQLAlchemy async or sync sessions with JSONB columns

#### Step 1.2 — Design PostgreSQL schema for document store

Each MongoDB collection maps to a PostgreSQL table with JSONB payload:

```sql
-- Generic document store table pattern (one per collection)
CREATE TABLE IF NOT EXISTS doc_chat_sessions (
    id TEXT PRIMARY KEY,           -- maps to _id
    data JSONB NOT NULL,           -- full document as JSONB
    session_id TEXT,               -- extracted index field
    user_id TEXT,                  -- extracted index field
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX IF NOT EXISTS idx_doc_chat_sessions_session_id ON doc_chat_sessions(session_id);
CREATE INDEX IF NOT EXISTS idx_doc_chat_sessions_user_id ON doc_chat_sessions(user_id);
CREATE INDEX IF NOT EXISTS idx_doc_chat_sessions_data ON doc_chat_sessions USING GIN(data);

-- Similar tables for: doc_agent_traces, doc_context_summaries, doc_archived_messages
-- Similar tables for: doc_llm_message_store, doc_session_ai_state
-- Similar tables for: doc_doc_summaries (ETL)
```

**Design decision:** Use JSONB `data` column for full document fidelity + extracted index columns for hot query paths. This preserves the schema-less nature while getting PostgreSQL's indexing and transactional guarantees.

#### Step 1.3 — Create `newui_test/core/doc_store.py`

Sync version of the abstraction (since `newui_test` uses `pymongo` sync, not `motor` async):

```
SyncDocStoreManager
  - Same interface but synchronous
  - _SyncMongoDocStore — wraps pymongo.MongoClient
  - _SyncPostgresDocStore — uses SQLAlchemy sessions
```

#### Step 1.4 — Create `etl/libs/doc_store.py`

ETL-specific version (sync, follows existing `etl/libs/vector_dbs/` pattern):

```
ETLDocStoreManager
  - _ETLMongoDocStore — wraps pymongo.MongoClient
  - _ETLPostgresDocStore — uses SQLAlchemy sessions
```

---

### Phase 2: Implement PostgreSQL Adapters

#### Step 2.1 — Implement `_PostgresDocStore` in `agentic/core/db/doc_store.py`

- Map `insert_one` → `INSERT INTO doc_<collection> (id, data) VALUES (:id, :data::jsonb)`
- Map `find_one` → `SELECT data FROM doc_<collection> WHERE data @> :filter::jsonb LIMIT 1`
- Map `find_many` → `SELECT data FROM doc_<collection> WHERE data @> :filter::jsonb ORDER BY ... LIMIT ...`
- Map `update_one` → `UPDATE doc_<collection> SET data = data || :update::jsonb WHERE data @> :filter::jsonb`
- Map `replace_one` → `INSERT ... ON CONFLICT (id) DO UPDATE SET data = :data`
- Map `delete_one/many` → `DELETE FROM doc_<collection> WHERE data @> :filter::jsonb`
- Map `aggregate` → translate common pipelines (`$group`, `$sort`, `$limit`) to SQL
- Map `count_documents` → `SELECT COUNT(*) FROM doc_<collection> WHERE data @> :filter::jsonb`
- Map `create_index` → `CREATE INDEX ... ON doc_<collection>(<field>)` or GIN index

#### Step 2.2 — Implement `_SyncPostgresDocStore` in `newui_test/core/doc_store.py`

Same mapping but using synchronous SQLAlchemy sessions (reusing `newui_test/core/db/connection.py`).

#### Step 2.3 — Implement `_ETLPostgresDocStore` in `etl/libs/doc_store.py`

Same mapping, reusing ETL's existing SQLAlchemy engine.

#### Step 2.4 — Create SQL migration scripts

- `agentic/database/doc_store_tables.sql` — Creates tables for `agentic` collections
- `newui_test/database/doc_store_tables.sql` — Creates tables for `newui_test` collections
- `etl/sql/doc_store_tables.sql` — Creates tables for ETL collections

---

### Phase 3: Migrate `agentic/` Repo

#### Step 3.1 — Refactor `memory/smart_memory_handler.py`

- Replace direct `motor` usage with `DocStoreManager` from `core/db/doc_store.py`
- `get_mongo_client()` → `get_doc_store()` (returns `DocStoreManager`)
- All collection accesses go through `doc_store.get_collection("chat_sessions")` etc.
- All CRUD operations use the unified interface
- `_ensure_mongo_indexes()` → `_ensure_doc_indexes()` (works for both backends)
- `close_mongo_connection()` → `close_doc_store_connection()`
- Keep function signatures identical so callers (`api/main.py`) don't change

#### Step 3.2 — Refactor `rag/rag_manager.py`

- Replace `_get_mongo_client()` with `DocStoreManager` calls
- Same interface, different backend dispatch

#### Step 3.3 — Update `agentic/api/main.py`

- Update imports if function names changed
- Update log messages ("MongoDB" → "document store")
- No structural changes needed if function signatures preserved

#### Step 3.4 — Update `agentic/core/config.py` (or `.env`)

- Add `DOC_STORE_BACKEND` env var (default: `mongodb` for backward compat)
- Add `DOC_STORE_PG_SCHEMA` env var (default: `public` or `doc_store`)

---

### Phase 4: Migrate `newui_test/` Repo

#### Step 4.1 — Refactor `core/database.py`

- `init_database()` → initialize `SyncDocStoreManager` instead of direct `MongoClient`
- `get_mongodb_collection()` → `get_doc_collection()` (returns `DocCollection`)
- `get_mongodb_db()` → `get_doc_store()` (returns `SyncDocStoreManager`)
- Keep backward-compat wrappers: `get_mongodb_collection()` delegates to new code

#### Step 4.2 — Refactor `services/message_service.py`

- `save_message()` → use `doc_store.insert_one("llm_message_store", doc)`
- `get_session_history()` → use `doc_store.find_many("llm_message_store", filter, sort, limit)`
- `get_all_sessions()` → translate aggregation pipeline to SQL equivalent:
  ```sql
  SELECT session_id, MAX(message) as last_message, MAX(timestamp) as last_timestamp,
         COUNT(*) as message_count, MIN(chatbot_id) as chatbot_id, MIN(user_id) as user_id
  FROM doc_llm_message_store
  GROUP BY session_id
  ORDER BY last_timestamp DESC
  ```

#### Step 4.3 — Refactor `services/session_ai_service.py`

- Replace `db[_COLLECTION]` calls with `doc_store` methods
- `replace_one` with `upsert=True` → `INSERT ... ON CONFLICT DO UPDATE`
- `find_one` → `SELECT data FROM ... WHERE data @> :filter`
- `update_one` → `UPDATE ... SET data = data || :update::jsonb WHERE ...`

#### Step 4.4 — Update `core/config.py`

- Add `DOC_STORE_BACKEND` env var
- Keep existing `MONGODB_*` vars for backward compat

---

### Phase 5: Migrate `etl/` Repo

#### Step 5.1 — Refactor `libs/base.py` (`HelixBase`)

- Replace `self.mongo = pymongo.MongoClient(...)` with `ETLDocStoreManager`
- Expose `self.doc_store` instead of `self.mongo`
- Keep `self.mongo` as a backward-compat property delegating to `_ETLMongoDocStore`

#### Step 5.2 — Refactor `my_works/connectors/MongoDBDocSummary.py`

- Replace direct `MongoClient` with `ETLDocStoreManager`
- All CRUD operations through unified interface

---

### Phase 6: Data Migration Script

#### Step 6.1 — Create `agentic/database/migrate_mongo_to_pg.py`

Standalone script that:
1. Connects to MongoDB (source)
2. Connects to PostgreSQL (target)
3. For each collection, reads all documents and inserts into corresponding PG table
4. Handles `_id` → `id` conversion
5. Handles datetime serialization
6. Supports `--dry-run` mode
7. Supports `--collection` flag to migrate specific collections only
8. Reports progress and counts

#### Step 6.2 — Create `newui_test/database/migrate_mongo_to_pg.py`

Same pattern for `newui_test` collections (`llm_message_store`, `session_ai_state`).

#### Step 6.3 — Create `etl/sql/migrate_mongo_to_pg.py`

Same pattern for ETL collections.

---

### Phase 7: Configuration & Docker

#### Step 7.1 — Update `.env` files

Add to `agentic/.env`, `newui_test/.env`, `etl/.env`:
```
DOC_STORE_BACKEND=postgresql  # or "mongodb" for backward compat
# PostgreSQL doc store uses existing POSTGRES_* connection vars
```

#### Step 7.2 — Update `setup/docker-compose.yml`

- Ensure PostgreSQL has the doc_store schema/tables created on startup
- Add init script or migration step to container entrypoint
- MongoDB service can remain for backward compat / migration period

#### Step 7.3 — Update `setup/.env.docker`

Add `DOC_STORE_BACKEND=postgresql` to Docker env config.

---

### Phase 8: Tests

#### Step 8.1 — Unit tests for `agentic/core/db/doc_store.py`

- `tests/test_doc_store.py`:
  - Test `DocStoreConfig` backend selection
  - Test `_MongoDocStore` with mocked motor client
  - Test `_PostgresDocStore` with mocked SQLAlchemy session
  - Test CRUD operations for both backends
  - Test `aggregate()` pipeline translation (group, sort, limit)
  - Test `create_index()` for both backends
  - Test `close()` cleanup

#### Step 8.2 — Unit tests for `newui_test/core/doc_store.py`

- `newui_test/tests/test_doc_store.py`:
  - Test sync CRUD operations for both backends
  - Test `get_all_sessions()` aggregation translation
  - Test session AI state pause/resume with PG backend

#### Step 8.3 — Integration tests for `agentic/`

- `tests/integration/test_doc_store_pg.py`:
  - Use testcontainers PostgreSQL (existing pattern from `conftest.py`)
  - Create doc_store tables, test real CRUD operations
  - Test `save_conversation()` / `get_conversation_history()` round-trip
  - Test `save_agent_trace()` / retrieval
  - Test `get_session_state_fields()` / `clear_session()`
  - Test concurrent access and transaction isolation

#### Step 8.4 — Integration tests for `newui_test/`

- `newui_test/tests/test_message_service_pg.py`:
  - Test `save_message()` / `get_session_history()` round-trip with PG
  - Test `get_all_sessions()` aggregation with PG
  - Test `session_ai_service` pause/resume with PG

#### Step 8.5 — Backward-compat tests

- Verify `DOC_STORE_BACKEND=mongodb` still works identically to current behavior
- Run existing test suite with both backends
- Test fallback behavior when PG is unavailable

#### Step 8.6 — Migration script tests

- `tests/test_migrate_mongo_to_pg.py`:
  - Test with small fixture data
  - Test `--dry-run` mode
  - Test idempotency (running twice doesn't duplicate)
  - Test `--collection` selective migration

---

### Phase 9: Documentation

#### Step 9.1 — Update `agentic/docs/`

- New: `agentic/docs/DOC_STORE_MIGRATION.md` — Architecture, config, schema, migration guide
- Update: `agentic/docs/ARCHITECTURE.md` — Add doc store layer to architecture overview
- Update: `agentic/README.md` — Add `DOC_STORE_BACKEND` to config section

#### Step 9.2 — Update `lat.md/` knowledge graph

- New section: `lat.md/architecture#Document Store` — describes the dual-backend doc store pattern
- Update: `lat.md/architecture#Database Layer` — cross-link to doc store
- Update: `lat.md/decisions` — ADR for MongoDB→PostgreSQL migration decision

#### Step 9.3 — Update `newui_test/` docs

- Update `newui_test/README.md` — Add doc store config section

#### Step 9.4 — Update `etl/` docs

- Update `etl/README.md` or `etl/NOTES.md` — Add doc store config

#### Step 9.5 — Update `.env.example` files

- `agentic/.env`, `newui_test/.env`, `etl/.env` — Add `DOC_STORE_BACKEND` with comments

---

### Phase 10: Rollout & Verification

#### Step 10.1 — Run migration script on staging

1. Set `DOC_STORE_BACKEND=mongodb` (current behavior, no change)
2. Run migration script to copy data to PostgreSQL
3. Verify row counts match
4. Switch `DOC_STORE_BACKEND=postgresql`
5. Run smoke tests (send a message, verify response, check session history)
6. Verify booking flow works end-to-end

#### Step 10.2 — Run full test suite

```bash
# Unit tests
cd agentic && python3 -m pytest tests/ -x -q

# Integration tests (requires Docker)
python3 -m pytest tests/integration/ -x -q

# Static analysis
python3 -m pytest tests/test_static_checks.py -x -q
```

#### Step 10.3 — lat.md validation

```bash
lat check  # All wiki links and code refs must pass
```

---

## Risk Mitigation

| Risk | Mitigation |
|---|---|
| MongoDB aggregation pipelines don't map cleanly to SQL | Only 1 aggregation pipeline exists (`get_all_sessions`); translate manually |
| Motor async → SQLAlchemy async mismatch | Use `sqlalchemy.ext.asyncio` async sessions for PG adapter in `agentic/` |
| Datetime serialization differences | Normalize all datetimes to ISO 8601 strings in JSONB |
| `_id` field type differences (ObjectId vs UUID) | Use string IDs throughout (already mostly string-based) |
| Existing tests break | Keep `DOC_STORE_BACKEND=mongodb` as default; run existing tests first |
| ETL `HelixBase.mongo` used by many subclasses | Keep backward-compat `self.mongo` property |

---

## File Change Summary

| Repo | Files to create | Files to modify |
|---|---|---|
| `agentic/` | `core/db/doc_store.py`, `database/doc_store_tables.sql`, `database/migrate_mongo_to_pg.py`, `tests/test_doc_store.py`, `tests/integration/test_doc_store_pg.py`, `docs/DOC_STORE_MIGRATION.md` | `memory/smart_memory_handler.py`, `rag/rag_manager.py`, `api/main.py`, `core/config.py` |
| `newui_test/` | `core/doc_store.py`, `database/doc_store_tables.sql`, `database/migrate_mongo_to_pg.py`, `tests/test_doc_store.py`, `tests/test_message_service_pg.py` | `core/database.py`, `core/config.py`, `services/message_service.py`, `services/session_ai_service.py` |
| `etl/` | `libs/doc_store.py`, `sql/doc_store_tables.sql`, `sql/migrate_mongo_to_pg.py` | `libs/base.py`, `my_works/connectors/MongoDBDocSummary.py` |
| `channel/` | — | — (no changes) |
| Root | — | `setup/docker-compose.yml`, `setup/.env.docker` |
| `lat.md/` | New sections | Update architecture, decisions |

---

## Execution Order

```
Phase 1 (Design)          → Step 1.1–1.4
Phase 2 (PG Adapters)     → Step 2.1–2.4
Phase 3 (agentic/)        → Step 3.1–3.4
Phase 4 (newui_test/)     → Step 4.1–4.4
Phase 5 (etl/)            → Step 5.1–5.2
Phase 6 (Migration script) → Step 6.1–6.3
Phase 7 (Config & Docker) → Step 7.1–7.3
Phase 8 (Tests)           → Step 8.1–8.6
Phase 9 (Docs)            → Step 9.1–9.5
Phase 10 (Rollout)        → Step 10.1–10.3
```

Each phase can be merged independently. Phases 3–5 can be done in parallel once Phase 2 is complete.

---

## Implementation Status (Updated)

### Completed

| Phase | Description | Files Created/Modified |
|---|---|---|
| 1 | Abstraction layers | `agentic/core/db/doc_store.py`, `newui_test/core/doc_store.py`, `etl/libs/doc_store.py` |
| 2 | SQL migration scripts | `agentic/database/doc_store_tables.sql`, `newui_test/database/doc_store_tables.sql`, `etl/sql/doc_store_tables.sql` |
| 3 | Refactor agentic/ | `agentic/memory/smart_memory_handler.py`, `agentic/rag/rag_manager.py` |
| 4 | Refactor newui_test/ | `newui_test/core/database.py`, `newui_test/services/message_service.py`, `newui_test/services/session_ai_service.py` |
| 5 | Refactor etl/ | `etl/libs/base.py`, `etl/my_works/connectors/MongoDBDocSummary.py` |
| 6 | Data migration scripts | `agentic/database/migrate_mongo_to_pg.py`, `newui_test/database/migrate_mongo_to_pg.py`, `etl/sql/migrate_mongo_to_pg.py` |
| 7 | Config & env files | `agentic/.env`, `newui_test/.env`, `setup/.env.docker` (added `DOC_STORE_BACKEND`) |
| 8 | Tests | `agentic/tests/test_doc_store_unit.py`, `agentic/tests/integration/test_doc_store_pg.py` |

### How to Switch Backends

1. **Keep MongoDB (default):** `DOC_STORE_BACKEND=mongodb` (or unset)
2. **Switch to PostgreSQL:** Set `DOC_STORE_BACKEND=postgresql` and run the SQL migration scripts:
   ```bash
   # For agentic:
   psql -f agentic/database/doc_store_tables.sql
   # For newui_test:
   psql -f newui_test/database/doc_store_tables.sql
   # For etl:
   psql -f etl/sql/doc_store_tables.sql
   ```
3. **Migrate existing data:**
   ```bash
   cd agentic && python database/migrate_mongo_to_pg.py --dry-run  # check counts
   cd agentic && python database/migrate_mongo_to_pg.py             # live migration
   cd newui_test && python database/migrate_mongo_to_pg.py
   cd etl && python sql/migrate_mongo_to_pg.py
   ```

### Running Tests

```bash
# Unit tests (no DB required):
cd agentic && python -m pytest tests/test_doc_store_unit.py -v

# Integration tests (requires Docker):
cd agentic && python -m pytest tests/integration/test_doc_store_pg.py -v -k postgres
```

