# Sector configuration

Controls which deterministic hard overrides and features are active per business sector, enabling sector-agnostic deployment.

## Configuration files

Two files define and load sector behavior at runtime.

- **`config/sector_config.json`** — JSON defining sectors, enabled overrides, and feature flags.
- **[[agentic/config/sector_loader.py]]** — Python loader with mtime caching. Exports `resolve_sector`, `is_override_enabled`, `is_feature_enabled`.

## Sector resolution

The active sector is resolved in priority order: `state["sector"]` from the API request, then `config["default_sector"]`, then `"booking"` as final fallback. The `sector` field is part of [[agentic/core/state.py]] `GraphState`.

## Defined sectors

Each sector has an `enabled_overrides` list and feature flags.

### booking

Full deterministic overrides for appointment-based businesses. All 12 overrides enabled, all feature flags active. Default sector for backward compatibility.

### generic

Sector-agnostic fallback. Only `business_info_rag`, `self_identity`, `assistant_identity` overrides enabled. All feature flags disabled. Relies on LLM classification for routing.

## Adding a new sector

Copy the `generic` entry in `sector_config.json`, rename it, and enable desired overrides.

Valid override names are listed in the `booking` sector. Unknown names are silently ignored. Feature flags (`enable_resource_markers`, `enable_owner_edit_detection`, `enable_listing_classifier`) activate booking-specific detection logic.

## Related

Links to related concepts and source files.

- [[concepts/supervisor]] — How hard overrides are applied in the intent classifier.
- [[agentic/workflow_builder/agents/intent_classifier.py]] — Gates each override via `is_override_enabled` and `is_feature_enabled`.
