# Design Decisions

Key architectural decisions, trade-offs, and rationale for the Agentic Platform.

## LangGraph for agent orchestration

**Decision:** Use LangGraph `StateGraph` with dynamic JSON-based graph construction instead of hard-coded agent chains.

**Rationale:** Organizations need different chatbot behaviors (some need booking + RAG, others just RAG, others custom workflows). JSON-defined graphs allow per-organization customization without code changes. The graph engine's `build_graph()` reads config, instantiates agents, and compiles the graph at request time.

**Trade-off:** Dynamic construction adds latency on first request per graph. Mitigated by keeping compiled graphs in memory and the relatively low cost of LangGraph compilation.

## Shared GraphState with reducers

**Decision:** Use a single `GraphState` TypedDict with custom reducers instead of separate state objects per subgraph.

**Rationale:** Booking confirmation flags (`pending_booking_confirmation`, etc.) need to persist across subgraph boundaries (e.g., a user says "yes" in what looks like a general turn but should confirm a pending booking). A shared state with `take_last` reducer for scalar fields and `merge_partial_answers` for multi-intent synthesis enables this.

**Trade-off:** The state dictionary is large and most fields are irrelevant to most agents. However, LangGraph's state slicing (each node only sees what it reads) mitigates this.

## Deterministic-first intent classification

**Decision:** Apply hard-coded keyword/regex rules before LLM classification for intent routing.

**Rationale:** Common patterns like bare confirmations ("yes"/"no" in booking context) and single-word service name follow-ups are trivially classifiable without an LLM call. This saves latency (~200-500ms) and cost on every such turn. The LLM path is the fallback for ambiguous or novel inputs.

**Trade-off:** The hard override rules must be maintained as new patterns emerge. Risk of misclassification if rules are too aggressive. Mitigated by scoping rules tightly (e.g., `pending_booking_confirmation` must be set).

## Two-tier owner edit disambiguation

**Decision:** Use a deterministic keyword/regex parser (`owner_edit_planner`) with LLM fallback (`owner_edit_llm`) for owner edit commands.

**Rationale:** Owner edits like "change phone to 555-1234" or "add service Haircut 30min 100TL" follow predictable patterns that a parser handles efficiently. Turkish morphological complexity (agglutination, vowel harmony) makes some cases ambiguous — the LLM fallback handles these.

**Trade-off:** Maintaining two code paths (parser + LLM) increases complexity. However, the parser handles ~80% of cases at near-zero latency, making the overall system faster and cheaper than LLM-only.

## Booking state via invisible message markers

**Decision:** Embed booking slot state as `[BOOKING_STATE:...]` JSON markers in `AIMessage.content` rather than using LangGraph's native state persistence.

**Rationale:** The booking flow spans multiple LLM calls within a single graph invocation. LangGraph state is updated between nodes but the LLM only sees `messages`. Embedding state in messages lets the LLM "see" the current slot progress without custom tool definitions. The markers are stripped before user display.

**Trade-off:** Fragile parsing if the LLM modifies the markers. Versioned format (v1/v2) and validation in `context_codec.py` mitigate this. The `BOOKING_STATE_WRITE_MODE` env var allows gradual rollout of format changes.

## PostgreSQL schemas over separate databases

**Decision:** Migrate from 3 MySQL databases to 1 PostgreSQL database with 3 schemas.

**Rationale:** PostgreSQL does not support cross-database queries. The application code uses `schema.table` notation extensively. By making `chat_bot` and `buyingen_2` schemas within `platform`, zero application code changes were needed.

**Trade-off:** Single database is a potential bottleneck and single point of failure. Mitigated by PostgreSQL's robust connection pooling and the fact that the three schemas serve different access patterns (transactional vs. config vs. analytics).

## MongoDB for conversations, PostgreSQL for configuration

**Decision:** Use MongoDB for chat messages and agent traces; PostgreSQL for organizational data, booking records, and workflow configs.

**Rationale:** Chat messages are append-heavy, schema-flexible documents that grow unbounded — MongoDB's document model fits naturally. Organizational data is relational (organizations → locations → resources → services → appointments) and benefits from PostgreSQL's ACID guarantees and JOINs.

**Trade-off:** Two database technologies to operate and maintain. The separation of concerns (operational vs. analytical vs. conversational data) justifies the operational cost.

## Multilingual config-driven design

**Decision:** Store all user-facing phrases and intent keywords in JSON config files, never hard-coded in Python.

**Rationale:** The platform supports Turkish and English. Config-driven phrases allow adding languages without code changes. The `booking_keywords.json` and `address_keywords.json` files serve as the single source of truth.

**Trade-off:** Config lookups add slight overhead vs. inline string constants. Acceptable given the i18n flexibility gained.

## Process-level caching with TTL

**Decision:** Use in-process dictionaries with TTL-based invalidation for organization details, service/resource name resolution, and entity lexicons.

**Rationale:** These lookups happen on every chat turn (resolve service name → ID, check organization timezone). Database round-trips would add 5-20ms per lookup. In-process caching with short TTLs (60s for org data, 300s for availability ranges) provides near-database freshness with sub-microsecond access.

**Trade-off:** Cache invalidation complexity — must explicitly clear caches on mutations (organization updates, slot bookings). The `invalidate_*` functions in `booking_manager.py` handle this.

## MongoDB to PostgreSQL Migration

**Decision:** Introduce a `DocStoreManager` abstraction layer that supports both MongoDB and PostgreSQL backends, selected at runtime via `DOC_STORE_BACKEND` env var, mirroring the existing [[concepts/doc-store|vector store dual-backend]] pattern.

**Rationale:** Operating two document databases (MongoDB + PostgreSQL) increases infrastructure complexity. PostgreSQL JSONB with GIN indexes provides equivalent schema-less query capability while consolidating to a single database instance. The dual-backend approach allows gradual migration without downtime — existing MongoDB deployments continue working, and switching to PostgreSQL is a config change plus a data migration script run.

**Trade-off:** The PostgreSQL adapter translates MongoDB-style filter operators (`$ne`, `$in`, `$gte`, etc.) to SQL JSONB queries, which adds a translation layer. Complex aggregation pipelines are replaced with Python-side processing for cross-backend compatibility. The trade-off is acceptable since the aggregation use cases (RAG reporting) are low-frequency and not latency-sensitive.

**Implementation:** Three abstraction layers were created — `DocStoreManager` (async, for agentic/), `SyncDocStoreManager` (sync, for newui_test/), and `ETLDocStoreManager` (sync, for etl/). Each delegates to a backend-specific adapter (`_MongoDocStore` or `_PostgresDocStore`). SQL migration scripts create `doc_<collection>` tables with JSONB data columns. Data migration scripts transfer existing MongoDB documents to PostgreSQL idempotently.
