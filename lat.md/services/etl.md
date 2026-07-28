# ETL Service

Document ingestion and processing pipeline. Handles file upload, chunking, embedding generation, and population of both ChromaDB vector collections and the `buyingen_2` PostgreSQL schema.

## Entry point

The application is defined in [[etl/main.py]]. It can run as a service or be invoked for individual document processing tasks.

## Key modules

Core directories for document processing, configuration, and data storage.

- `etl/libs/` — Core processing libraries for document parsing, chunking, and embedding.
- `etl/config/` — Configuration for document sources, chunking strategies, and embedding models.
- `etl/migrations/` — Database migrations for the `buyingen_2` schema.
- `etl/scripts/` — Utility scripts for batch processing and data maintenance.
- `etl/docs/` — Documentation for the ETL pipeline.

## Document processing pipeline

Five-stage pipeline from raw file upload to searchable vector embeddings and database records.

1. **Ingestion** — Files uploaded to `etl/uploads/` or pulled from Google Drive.
2. **Parsing** — Extract text from PDF, DOCX, and other formats.
3. **Chunking** — Split documents into semantic chunks with configurable size and overlap.
4. **Embedding** — Generate vector embeddings using OpenAI `text-embedding-3-small`.
5. **Storage** — Store chunks in both ChromaDB (for vector search) and PostgreSQL `buyingen_2` schema (for metadata and source tracking).

## Integration with agentic

The ChromaDB collections populated by ETL are queried by the agentic RAG subgraph at runtime. See [[concepts/rag]] for the retrieval flow.
