# Database Architecture

The platform uses a single PostgreSQL database (`platform`) with three schemas, migrated from MySQL. MongoDB is used for conversation storage and agent traces.

## PostgreSQL: `platform` database

Connection: `postgresql+psycopg2://postgres:mysecret@localhost:5433/platform`

### Schema: `public` (50+ tables)

Core application tables for organization management, booking, workflows, and configuration:

| Table | Purpose |
|---|---|
| `users` | User accounts and authentication |
| `booking_organizations` | Organization profiles with unique `ORG-xxx` IDs |
| `booking_locations` | Physical locations per organization with `LOC-xxx` IDs |
| `booking_resources` | Staff/experts available for appointments |
| `booking_services` | Services offered by organizations |
| `booking_services_settings` | Links services to resources (makes them bookable) |
| `booking_availability_default_settings` | Default working hours per organization type |
| `booking_availability_organization_settings` | Per-organization working hours overrides |
| `booking_availability_settings` | Per-resource working hours (the actual schedule) |
| `booking_appointments` | Booked appointments |
| `workflows` / `workflow_*` | LangGraph workflow configurations |
| `agent_prompts` | Customizable agent prompt templates |
| `llm_configs` | LLM provider and model configurations |
| `organizations` | Multi-tenant organization metadata |
| `companies` | Company profiles linked to organizations |

### Schema: `chat_bot` (66 tables)

Chatbot configuration and messaging:

| Table | Purpose |
|---|---|
| `chatbots` | Chatbot instances linking user, org, graph, and collection |
| `collections` | ChromaDB collection references |
| `users` | Chatbot end-user profiles |
| `message_sessions` | Active conversation sessions |
| `companies` | Company/business profiles |
| `chatbot_user_interactions` | User interaction tracking |

### Schema: `buyingen_2` (12 tables)

ETL pipeline and document storage:

| Table | Purpose |
|---|---|
| `chunks` | Document chunks with embeddings metadata |
| `sources_*` | Source document tracking |
| `chatbot_user_interactions` | ETL-scoped interaction tracking |

### pgvector tables (public schema)

Vector embeddings stored directly in PostgreSQL via the `pgvector` extension:

| Table | Purpose |
|---|---|
| `langchain_pg_collection` | Vector collection metadata (name, UUID, cmetadata) |
| `langchain_pg_embedding` | Document embeddings (collection_id, embedding vector(1536), document, cmetadata) |

These are the standard langchain-postgres schema tables. A GIN index on `cmetadata` enables JSONB containment queries for metadata filtering. The `vector` extension must be enabled: `CREATE EXTENSION IF NOT EXISTS vector`.

The vector store uses a dual-backend abstraction in [[agentic/core/db/vector_store.py#VectorStoreManager]]. pgvector is the **default backend** (`VECTOR_STORE_BACKEND=pgvector`), with ChromaDB as a fallback. A migration script at `newui_test/scripts/migrate_chroma_to_pgvector.py` handles bulk transfer from ChromaDB.

## Why schemas instead of separate databases?

MySQL allows cross-database queries with `database.table` syntax. PostgreSQL does not. By migrating `chat_bot` and `buyingen_2` as schemas within `platform`, the existing `schema.table` query syntax works without application code changes.

## MySQL to PostgreSQL migration

The migration was performed by scripts in `agentic/scripts/`:
- [[agentic/scripts/migrate_mysql_to_pg.py]] — Migrates the `platform` database (public schema).
- [[agentic/scripts/migrate_aux_dbs.py]] — Migrates `chat_bot` and `buyingen_2` as schemas.

Key type mappings:
- `tinyint(1)` → `SMALLINT` (boolean flags)
- `enum(...)` → `VARCHAR(50)` (app validates values)
- `longtext + json_valid()` → `JSONB` (JSON columns)
- `AUTO_INCREMENT` → `SERIAL`/`BIGSERIAL` (with sequence sync)
- MySQL zero dates `'0000-00-00 00:00:00'` → `NULL` or epoch

## MongoDB

Used for conversation storage and operational data:

| Collection | Purpose |
|---|---|
| `chat_sessions` | Chat messages per session |
| `agent_traces` | Agent execution traces |
| `context_summaries` | LLM-generated conversation summaries |
| `archived_messages` | Messages evicted from active context |

See [[concepts/memory]] for the memory system design.

## Vector store (pgvector + ChromaDB)

Dual-backend vector storage with **pgvector as the default** and ChromaDB as a fallback. The [[agentic/core/db/vector_store.py#VectorStoreManager]] abstraction selects the active backend via the `VECTOR_STORE_BACKEND` env var.

When using pgvector, embeddings live in `langchain_pg_collection` / `langchain_pg_embedding` tables within the `platform` database. When using ChromaDB, collections are stored on disk in `agentic/chroma_phrase_db/`. Embeddings are generated with OpenAI `text-embedding-3-small` (dimension 1536).

See [[concepts/rag#Vector store]] for the full dual-backend design and migration details.
