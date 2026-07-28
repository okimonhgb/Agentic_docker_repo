# Channel Service

External messaging channel service for WhatsApp Business integration. A FastAPI application (port 8010) that receives webhooks from messaging providers, normalizes incoming messages, and forwards them to the newui_test chat API.

The service is designed to be a lightweight bridge — it does not contain business logic, only message translation and routing.

## Entry point

The application is defined in [[channel/app/main.py]] with a lifespan handler that initializes providers and warms up the database connection.

## Provider architecture

Uses a registry pattern to support multiple messaging providers:
- [[channel/app/core/registry.py]] — Provider initialization and registration.
- `channel/app/providers/` — Provider-specific implementations (Chakra WhatsApp, Meta WhatsApp).

Each provider handles:
- Webhook verification (HMAC signature validation)
- Message parsing (text, media, location, interactive)
- Message normalization to a common internal format
- Response delivery back through the provider's API

## API endpoints

HTTP endpoints for receiving webhooks, serving the admin dashboard, and health checks.

- `POST /webhooks/{provider}` — Incoming webhook from a messaging provider. Validates, normalizes, and forwards to newui_test.
- `GET /webhooks/{provider}` — Webhook verification (challenge-response) for provider registration.
- `GET /admin/` — Simple admin dashboard for monitoring.
- `GET /health` — Health check.

## Configuration

Settings are loaded from environment variables via [[channel/app/config.py]]. Provider credentials (API keys, webhook secrets) are configured per provider.
