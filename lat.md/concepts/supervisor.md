# Supervisor and Intent Routing

The top-level orchestrator that classifies every user message and routes it to subgraph(s). Uses deterministic rules (gated by [[concepts/sector-config|sector config]]) with LLM fallback.

## Intent classifier

The entry point for every chat turn. Defined in [[agentic/workflow_builder/agents/intent_classifier.py]].

### Word-boundary-safe keyword matching

The `_safe_keyword_in_query` helper prevents false substring matches for short keywords (≤4 chars) using word-boundary regex.

Longer keywords (5+ chars) use plain substring check. All keyword signal computations in intent classifier, router agent, and booking intent agent use this helper.

### Sector configuration

[[concepts/sector-config]] defines which hard overrides and features are active per sector.

The active sector is resolved from `state["sector"]` or defaults to `"booking"`. The `"generic"` sector disables all booking-specific overrides, relying on LLM classification.

### Hard overrides (deterministic, no LLM call)

Applied in order before any LLM classification. Each override is gated by `is_override_enabled(sector, override_name)`:
1. **Confirmation in booking context** (`confirmation_booking_context`) — Bare "yes/no/evet/hayır" when `pending_booking_confirmation` or `pending_cancel_confirmation` is set → routes to booking subgraph directly.
2. **Pending owner edit** (`owner_edit`) — When an owner edit is awaiting confirmation → routes to booking subgraph. Also gated by `enable_owner_edit_detection` feature flag. Exception: during an active pending booking confirmation/reschedule, a resource-prefixed reply (e.g. "dr. hepsi olsun") is a customer expert-switch follow-up, not an owner edit — it falls through to the booking subgraph's resource-prefix handling.
3. **Expert list query** (`expert_list`) — Resource nouns + name markers detected → routes to booking subgraph. Gated by `enable_resource_markers` feature flag.
4. **Expert availability query with date** (`expert_availability_with_date`) — A query with resource nouns, question words, and a date reference → routes to booking subgraph.
5. **Service list query** (`service_list`) — Service noun + question word detected → routes to RAG (or booking+RAG if mixed with address/product signals).
6. **Booking-first** (`booking_first`) — Strong booking signals (appointment keywords, booking priority keywords) → routes to booking (or booking+RAG if mixed with business-info signals).
7. **Business-info RAG** (`business_info_rag`) — Pure business-info queries (address, phone, hours) without booking intent → routes to RAG only.
8. **Self-identity** (`self_identity`) — "who am I" / "ben kimim" queries → routes to general.
9. **Assistant identity** (`assistant_identity`) — "who are you" / "sen kimsin" queries → routes to general.
10. **List booking only** (`list_booking_only`) — Appointment list queries ("randevularım", "my appointments") → routes to booking only, never RAG.
11. **Booking followup — needs clarification** (`booking_followup_needs_clarification`) — Short follow-up while booking clarification is pending → routes to booking.
12. **Booking followup — short** (`booking_followup_short`) — Short follow-up after recent booking-request language → routes to booking.

### Booking subgraph listing classifier suppressions

Within the booking subgraph's intent agent ([[agentic/agents/booking/intent_agent.py]]), the LLM listing classifier can categorize queries as `LIST_SERVICES`, `LIST_EXPERTS`, or `LIST_EXPERT_SERVICES`. Two deterministic suppressions prevent misclassification:
- **LIST_EXPERTS with date reference** — Suppressed so the query falls through to the normal BOOK/CHECK flow for service clarification.
- **LIST_EXPERT_SERVICES with booking request** — Suppressed when the query contains appointment markers (e.g. "randevu alabilirim") so availability info is provided instead of a plain service listing.

### LLM classification

If no hard override matches, the classifier calls an LLM with structured output to produce an `IntentOutput` containing one or more of: `booking`, `rag`, `general`.

## Multi-intent routing

[[agentic/workflow_builder/agents/routers.py]] provides:
- `route_to_subgraph` — Routes a single intent to its matching subgraph.
- `route_to_subgraphs` — For multiple intents, uses LangGraph's `Send` API to execute subgraphs in parallel, then routes to the synthesizer.
- `route_after_subgraph` — After subgraph completion, routes to synthesizer (if more subgraphs pending) or END.

## Synthesizer

[[agentic/workflow_builder/agents/synthesizer.py]] merges `partial_answers` from parallel subgraph executions into one coherent, non-redundant reply. It handles deduplication of overlapping information (e.g., if both RAG and booking mention the same service) and natural language merging.

## General response

[[agentic/workflow_builder/agents/general_response.py]] handles queries outside booking and RAG scope: chit-chat, identity questions ("who am I?"), and general knowledge. Uses a lightweight LLM call without tool binding.

## Supervisor node factory

[[agentic/toolbox/agents/supervisor.py#create_supervisor_node]] is a generic, reusable node factory for creating supervisor-style routing nodes. It takes a team list, output model, and optional prompt override. Used by the graph builder to construct supervisor nodes dynamically from JSON config.
