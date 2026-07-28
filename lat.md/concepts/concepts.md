# Domain Concepts

Core business logic and architectural patterns of the Agentic Platform.

- [[booking]] — Appointment booking domain: slot state machine, availability, owner edits
- [[rag]] — Retrieval-Augmented Generation: document ingestion, vector search, grounded answers
- [[graph-engine]] — Dynamic LangGraph graph construction from JSON configuration
- [[memory]] — Multi-tier conversation memory with summarization and agent traces
- [[workflow-builder]] — LLM-driven workflow generation from natural language descriptions
- [[supervisor]] — Intent classification and routing across booking, RAG, and general subgraphs
- [[doc-store]] — Dual-backend document storage (MongoDB/PostgreSQL) for sessions, traces, and messages
