# PlatformUI Service

Admin dashboard and organization management UI. A Flask application that provides the web interface for organization owners to create and manage their chatbot applications, configure booking settings, manage services and resources, and view reports.

## Entry point

The application is started via [[platformui/app.py]] which creates the Flask app with extensions (SQLAlchemy, Babel, etc.).

## Key modules

The main functional areas of the platform UI, organized by responsibility.

### App creation

Orchestrates the full organization and chatbot bootstrap flow including DB records and workflow registration.

- [[platformui/app_creator.py#create_complete_app]] — Orchestrates the full organization + chatbot creation flow: generates unique IDs, creates database records, sets up ChromaDB collections, registers the LangGraph workflow with agentic, and sends notifications.

### Booking management

Admin-facing booking CRUD, availability management, and RAG synchronization.

- [[platformui/booking_api.py]] — REST API for booking CRUD operations from the admin UI.
- [[platformui/booking_routes.py]] — Flask blueprint for booking-related pages and endpoints.
- [[platformui/booking_authorization.py]] — Authorization checks for booking operations.
- [[platformui/booking_rag_sync.py]] — Synchronizes booking resource/service documents with RAG vector collections.
- [[platformui/simple_availability_manager.py]] — Availability calendar and slot management for admin.
- [[platformui/remote_booking_manager.py]] — Remote booking operations forwarded to agentic.

### Authentication

User login, session handling, role-based access, and OAuth SSO integration.

- [[platformui/auth.py]] — User authentication and session management.
- [[platformui/authorization.py]] — Role-based access control.
- [[platformui/famethink_auth.py]] — OAuth integration for Famethink SSO.

### Chatbot configuration

Database and configuration operations for chatbot settings, LLM models, and search.

- [[platformui/chatbot_db_lib.py]] — Database operations for chatbot configuration (PostgreSQL).
- [[platformui/chatbot_db_mongo_lib.py]] — Database operations for conversation data (MongoDB).
- [[platformui/chatbot_llm_lib.py]] — LLM configuration management.
- [[platformui/chatbot_search_lib.py]] — Search configuration for chatbots.

### Notifications and reporting

Email dispatch, SMTP configuration, and usage analytics endpoints.

- [[platformui/send_notifications.py]] — Email notification dispatch.
- [[platformui/mail_utils.py]] — Email composition and SMTP utilities.
- [[platformui/reporting_routes.py]] — Usage and analytics reporting endpoints.

### User interaction tracking

Tracks end-user behavior and interaction events for analytics.

- [[platformui/user_interactions.py]] — Tracks user behavior and interaction events.

### Session management

Admin tools for managing active user sessions.

- [[platformui/manage_sessions.py]] — Admin session management.
- [[platformui/session_utils.py]] — Session utility functions.

### Workflow management

Routes for configuring and managing LangGraph workflow definitions.

- [[platformui/workflow_routes.py]] — Workflow configuration routes for admin.

## Database

Uses SQLAlchemy with PostgreSQL. The ORM models are defined in [[platformui/models.py]]. Database migrations are managed via Flask-Migrate/Alembic in `platformui/migrations/`.

The platformui connects to the same PostgreSQL `platform` database as agentic and newui_test, sharing the `public`, `chat_bot`, and `buyingen_2` schemas. See [[data/database]].
