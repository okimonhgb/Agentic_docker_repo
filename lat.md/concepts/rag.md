# RAG (Retrieval-Augmented Generation)

The RAG domain provides document-grounded question answering. Users can ask questions about an organization's knowledge base, and the system retrieves relevant document chunks from ChromaDB, then generates answers grounded in those documents.

## RAG subgraph agents

The RAG workflow is a LangGraph subgraph defined by agents in `agentic/agents/rag/`. State is defined in [[agentic/rag/state.py#RAGState]].

### Intake agent
Initializes RAG-specific state for the current turn. Defined in [[agentic/agents/rag/intake_agent.py]].

### Router agent
Classifies the user query into one of three paths: direct chat (no retrieval needed), clarify (ambiguous query), or retrieve (perform vector search). Defined in [[agentic/agents/rag/router_agent.py]].

### Chat agent
Answers directly without document retrieval — for small talk or questions answerable from conversation context. Defined in [[agentic/agents/rag/chat_agent.py]].

### Clarify agent
Asks a clarifying question when the user's query is too ambiguous for effective retrieval. Defined in [[agentic/agents/rag/clarify_agent.py]].

### Rewrite query agent
Optimizes the user's raw query for better retrieval performance — expands abbreviations, resolves pronouns, adds context. Defined in [[agentic/agents/rag/rewrite_query_agent.py]].

### Transform query agent
Generates alternative query formulations to improve recall. Defined in [[agentic/agents/rag/transform_query_agent.py]].

### Retrieve agent
Performs ChromaDB vector similarity search against the organization's document collection. Defined in [[agentic/agents/rag/retrieve_agent.py]].

### Grade documents agent
Assesses retrieved document relevance. Defined in [[agentic/agents/rag/grade_documents_agent.py]]. Sets `docs_quality` to "ok" or "bad".

### Check answer agent
Validates that the generated answer is grounded in the retrieved documents (hallucination check). Defined in [[agentic/agents/rag/check_answer_agent.py]].

### Answer RAG agent
Generates the final grounded answer from retrieved documents. Defined in [[agentic/agents/rag/answer_rag_agent.py]]. Injects timezone-aware current date and time into the system prompt so the LLM can answer date/time questions (e.g., "what time is it now?").

### Response agent
Formats the final answer for user display. Defined in [[agentic/agents/rag/response_agent.py]].

## RAG routing

The RAG subgraph has internal routing logic defined in [[agentic/agents/rag/routers.py]]:
- `route_by_intent` — Routes to chat, clarify, or retrieve based on router agent output.
- `route_by_quality` — Routes to answer generation or retry/clarify based on document quality.
- `route_by_decision` — Final decision: done, retry retrieval, or ask for clarification.

## RAG state

[[agentic/rag/state.py#RAGState]] extends `GraphState` with:
- `route` — Current routing decision (chat/rag/clarify).
- `rewritten_query`, `transformed_query` — Query transformations.
- `documents` — Retrieved LangChain Document objects.
- `docs_quality` — Quality assessment (ok/bad).
- `decision` — Final routing decision (done/retry/clarify).
- `attempts` — Loop control counter.

## RAG manager

[[agentic/rag/rag_manager.py]] provides the operational interface for RAG: collection management, document indexing, and retrieval configuration.

## Vector store

Vector storage uses a dual-backend architecture with **pgvector as the default**. The [[agentic/core/db/vector_store.py#VectorStoreManager]] abstraction dispatches to the configured backend at runtime via the `VECTOR_STORE_BACKEND` env var (default: `"pgvector"`).

### pgvector (default)

Embeddings stored in PostgreSQL via the `pgvector` extension, using standard langchain-postgres schema tables.

The `_PGVectorManager` adapter in `vector_store.py` uses `langchain-postgres.PGVector` for vector operations and direct SQL via SQLAlchemy for collection management (list, delete, stats).

Metadata filtering uses PostgreSQL JSONB `@>` containment queries with a GIN index on `cmetadata`. The `vector` extension must be enabled (`CREATE EXTENSION IF NOT EXISTS vector`). Embedding dimension is 1536 (OpenAI `text-embedding-3-small`).

### ChromaDB (fallback)

When `VECTOR_STORE_BACKEND=chromadb`, the `_ChromaManagerAdapter` delegates to the original ChromaDB manager via `langchain-chroma`. Collections are stored on disk in `agentic/chroma_phrase_db/` or accessed via HTTP API for remote instances.

### Migration

A migration script at `newui_test/scripts/migrate_chroma_to_pgvector.py` handles bulk transfer from ChromaDB to pgvector, with batch import, dry-run mode, and `--collection` filtering.

## RAG sync

[[agentic/booking/rag_sync_hooks.py]] ensures booking resource/service documents stay synchronized with RAG collections. When an owner edits a service description or adds a new resource, the corresponding vector store documents are updated via the configured backend.
