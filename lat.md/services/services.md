# Services

Microservices that compose the Agentic Platform.

- [[agentic]] — Core LangGraph-based agentic workflow engine (FastAPI, port 8008)
- [[newui-test]] — Chat API frontend service handling user sessions and message routing (FastAPI, port 8000)
- [[platformui]] — Admin dashboard and organization management UI (Flask)
- [[channel]] — External messaging channel for WhatsApp integration (FastAPI, port 8010)
- [[etl]] — ETL pipeline for document ingestion, embedding, and knowledge base population
