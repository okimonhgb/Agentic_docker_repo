# Booking Domain

The booking domain handles appointment scheduling, availability management, rescheduling, cancellations, and owner administrative operations — all through natural language conversation.

## Slot state machine

The core abstraction for tracking booking progress through a conversation turn. Defined in [[agentic/booking/slot_state_machine.py]].

### Slot states

The five stages a booking slot progresses through during a conversation turn.

- **EMPTY** — No booking information has been captured yet.
- **PARTIAL** — Some fields are filled (e.g., service selected but no date/time).
- **READY** — All required fields populated (service, date, time); ready for confirmation.
- **PENDING_CONFIRMATION** — Slot is complete, awaiting user "yes/no" confirmation before committing.
- **COMMITTED** — Appointment has been persisted to the database.

### Slot context

The [[agentic/booking/slot_state_machine.py#SlotContext]] dataclass tracks: `service_id`, `service_name`, `resource_id`, `resource_name`, `slot_date`, `slot_start`, `slot_end`. The machine enforces a logical progression: service → date → time → confirm → commit.

## Context codec

Booking state is embedded in conversation messages as invisible markers to persist slot progress across LLM calls without polluting the visible chat. Defined in [[agentic/booking/context_codec.py]].

State markers are JSON strings prefixed with `[BOOKING_STATE:` (v1) or `[BOOKING_STATE_V2:` (v2) and injected into `AIMessage.content`. The v2 format supports richer context and is gated by `BOOKING_STATE_WRITE_MODE` env var.

Key functions:
- [[agentic/booking/context_codec.py#parse_booking_state_content]] — Extracts and decodes booking state from message content.
- [[agentic/booking/context_codec.py#is_booking_state_marker_content]] — Detects if message content is a state marker (used to filter from user-facing display).

## Booking subgraph agents

The booking workflow is a LangGraph subgraph with specialized agents, each handling one responsibility:

### Intent agent
Classifies the booking-specific action from user input. Defined in [[agentic/agents/booking/intent_agent.py]].
Categories: chitchat, check availability, book, reschedule, cancel, list appointments, list experts/services, admin block time, owner edit.

### Datetime agent
Extracts and normalizes date/time expressions from natural language (Turkish and English). Defined in [[agentic/agents/booking/datetime_agent.py]].

Handles relative dates ("next Monday"), ranges ("this week"), and time preferences ("morning", "after 3pm"). When the LLM-resolved date is beyond the booking horizon, it clamps to `max_date` instead of rejecting, so the availability agent can still show the nearest slots.

### Availability agent
Queries the database for available slots matching the requested service, resource, date, and time. Defined in [[agentic/agents/booking/availability_agent.py]].

When the requested date is beyond the organization's booking horizon (max available slot date), the agent **clamps** the date to `max_date` and proceeds with the query, rather than rejecting it. This ensures users always see the nearest available slots.

### Negotiation agent
When the requested slot is unavailable, proposes alternative nearby slots. Defined in [[agentic/agents/booking/negotiation_agent.py]].

### Booking agent
Performs the actual create/replace/cancel appointment mutation against the database. Defined in [[agentic/agents/booking/booking_agent.py]]. Handles PostgreSQL operations for booking CRUD.

### List appointments agent
Retrieves and formats a user's upcoming appointments. Defined in [[agentic/agents/booking/list_appointments_agent.py]].

### List info agent
Lists available experts (resources) and services for an organization. Defined in [[agentic/agents/booking/list_experts_agent.py]].

### Admin availability agent
Allows organization owners to block or unblock time ranges via chat. Defined in [[agentic/agents/booking/admin_availability_agent.py]].

### Owner edit agent
Allows owners to edit contact info, services, and staff via chat with a preview → confirm → apply flow. Defined in [[agentic/agents/booking/owner_edit_agent.py]].

### Response agent
Common terminal node that composes the final user-facing text from whichever upstream agent ran. Defined in [[agentic/agents/booking/response_agent.py]].

The `_booking_state_passthrough()` helper spreads booking state fields into every return dict, ensuring they survive the subgraph-to-parent-graph state merge. It includes `pending_booking_confirmation`, `pending_cancel_confirmation`, `pending_reschedule`, and other slot/clarification fields. Without this passthrough, LangGraph subgraph state merging can drop these flags, causing the bot to lose confirmation context between turns (e.g., user says "evet" but the bot re-shows availability instead of finalizing the booking).

## Owner edit planning

Owner edits use a two-tier disambiguation strategy:

### Deterministic planner
[[agentic/booking/owner_edit_planner.py]] — Fast keyword/regex-based parser that handles clear-cut edit commands (e.g., "change phone to 555-1234"). Covers the majority of cases without an LLM call.

### LLM fallback
[[agentic/booking/owner_edit_llm.py]] — Invoked when the deterministic parser cannot confidently resolve the edit type (staff vs. contact vs. service disambiguation), especially for morphologically complex Turkish inputs.

## Booking manager

The [[agentic/booking/booking_manager.py]] module is the database access layer for all booking operations. It contains:
- Service and resource name resolution with process-level caching.
- Organization detail caching with configurable TTL.
- Availability date range computation and caching.
- Entity lexicon caching for name detection in user messages.
- Slot querying against `booking_availability_settings` and `booking_appointments`.
- Timezone-aware past-time filtering using `ZoneInfo("Europe/Istanbul")` (not container UTC) so slot availability matches the user's local time.
- Dialect-aware SQL via `BaseRepository` helpers (`group_concat`, `curdate`, `date_sub_days`) for MySQL/PostgreSQL compatibility.

All "today"/"current time" comparisons (real slot filtering, min/max availability range, and virtual gap-slot synthesis in `_generate_gap_slots`) go through [[agentic/booking/booking_manager.py#BookingManager#_local_now]], a shared helper returning timezone-aware `datetime.now(ZoneInfo(...))`. Using the container's naive UTC clock instead would show/hide "today" slots at the wrong wall-clock time relative to the user's timezone.

## Organization management

[[agentic/booking/organization_manager.py]] handles CRUD operations for booking organizations, locations, services, and resources. Called by both the owner edit agent and the platformui admin.

## RAG integration for booking

Bridges booking data into the RAG system so users can ask informational questions about services.

[[agentic/booking/rag_integration.py]] bridges booking data with the RAG system. Service descriptions, resource profiles, and organization documents are indexed as ChromaDB documents so users can ask informational questions ("what services do you offer?") that may be answered via RAG rather than structured booking queries.

## State propagation refactor plan

A staged plan to eliminate the dual state propagation problem is in `agentic/docs/BOOKING_STATE_REFACTOR_PLAN.md`.

Empirical verification with LangGraph 1.0.2 showed `Command(update=...)` on shared GraphState keys propagates correctly from subgraphs — the hidden-marker workaround premise is stale. The plan promotes cross-turn booking fields into GraphState (the shared schema), then deletes the marker channel, cleans up guards and return paths, and improves trace readability (agent attribution, dedup, real timestamps).

## Booking state in GraphState

Booking-specific fields are defined directly in the shared [[agentic/core/state.py#GraphState]] (not a separate sub-state) to enable cross-subgraph persistence. Key fields:
- `pending_reschedule`, `pending_booking_confirmation`, `pending_cancel_confirmation` — Confirmation flow gates.
- `slot_date`, `slot_start`, `slot_end`, `slot_timezone` — Current slot being worked on.
- `service_id`, `service_name`, `resource_id`, `resource_name` — Selected service/resource.
- `old_slot_date`, `old_slot_start`, `old_resource_id` — Previous slot for reschedule/cancel.
