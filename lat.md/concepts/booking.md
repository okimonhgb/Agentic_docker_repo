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
Classifies booking-specific intent. Alternative-seeking follow-ups ("başka ne zamanlar", "diğer uygun tarihler") are forced to CHECK range queries instead of confirmations, so the bot does not re-offer the same single date.

Three hard overrides improve intent classification accuracy:

- **Reschedule without appointment marker**: When the user has existing booking context and uses a reschedule verb (e.g. "saat 16ya değiştirir misin") without explicitly saying "randevu", the intent is forced to RESCHEDULE instead of falling through to the LLM classifier which may route to CHECK/BOOK.
- **BOOK override for booking verbs**: When the user says "randevu alır mısın" or similar (appointment marker + booking verb, no reschedule marker), the intent is forced to BOOK — preventing the LLM from misclassifying as RESCHEDULE when existing booking context is present.
- **Bare hour detection**: Short replies like "16" after the bot asks "Hangi saat sizin için uygun?" are now detected as explicit time signals and routed to BOOK with the carried date, preventing the LLM from picking up stale booking dates.
- **RESCHEDULE after LIST**: When the user has just listed their appointments and says "saat 14e alır mısın" (booking verb + time, no "randevu" or "yeni"), the intent is forced to RESCHEDULE since they're referring to their existing appointment, not booking a new one.
- **BOOK confirmation priority**: When `pending_booking_confirmation` is True, an affirmative "evet" always routes to BOOK — even if stale `old_slot_date`/`old_slot_start` from a previous RESCHEDULE persist in state (via `take_last_not_none` reducer). Without this, the bot would incorrectly execute a RESCHEDULE instead of creating a new booking.
- **Post-LLM BOOK→RESCHEDULE override**: When the LLM classifier returns BOOK but the user used a possessive appointment marker (e.g. "randevumu", "my appointment") right after listing their appointments (LIST intent), the intent is overridden to RESCHEDULE. This prevents duplicate bookings when the user says "9 daki randevumu manikür yap" (make my 9 o'clock appointment manikür) — they're modifying an existing appointment, not creating a new one.
- **Service-only change**: When a RESCHEDULE has the same old and new date/time but a different service_name, the booking agent detects it as a service-only change. The new service_id is resolved from service_name and passed to `reschedule_booking`, which updates the service on the existing booking slots. The datetime_agent prioritizes the user's explicitly mentioned service_name over the old booking's service.
- **Relative date/time resolution in RESCHEDULE**: When the user says "1 gün sonraya", "2 hafta önce", "1 saat sonraya", or "30 dakika önce", the datetime_agent deterministically resolves the new date/time relative to `old_slot_date`/`old_slot_start` (not today). Supports day, week, hour, and minute units in Turkish (gün, hafta, saat, dakika) and English (day, week, hour, minute). Handles midnight crossings for hour/minute offsets. The LLM often resolves relative dates from today's date, producing the wrong target date when the user means "1 day after my appointment".
- **Clearing pending flags on non-transactional intents**: When the intent is LIST, CHITCHAT, or CHECK (without active clarification context), `pending_reschedule`, `pending_booking_confirmation`, and `pending_cancel_confirmation` are set to `False`. Without this, a stale `pending_reschedule=True` from a previous confirmation prompt would cause the next RESCHEDULE request to skip confirmation and execute directly.

### Datetime agent
Extracts and normalizes date/time expressions from natural language (Turkish and English). Defined in [[agentic/agents/booking/datetime_agent.py]].

Handles relative dates ("next Monday"), ranges ("this week"), and time preferences ("morning", "after 3pm"). When the LLM-resolved date is beyond the booking horizon, it clamps to `max_date` instead of rejecting, so the availability agent can still show the nearest slots.

Discards stale past `slot_date` values from state snapshots and BOOKING_STATE markers — if the persisted date is before today, it is cleared so fresh availability queries run instead of failing with "past date" errors.

Detects follow-ups that explicitly ask for alternative dates/times ("başka ne zamanlar", "diğer tarihler", "farklı saatler", "other times"). These clear any extracted or carried `slot_date`/`slot_start` and force `is_range_query=True`, so the user sees the full availability window instead of being stuck on a single previously-offered date.

Also exposes two small deterministic helpers, [[agentic/agents/booking/datetime_agent.py#_is_alternative_seeking_query]] and [[agentic/agents/booking/datetime_agent.py#_is_open_availability_question]], so the classification is unit-testable rather than buried inside the LLM extraction node. Tests include [[agentic/tests/test_booking_state_fixes.py#test_alternative_seeking_turkish_diger_uygun_zamanlar]] and [[agentic/tests/test_booking_state_fixes.py#test_open_availability_turkish_müsaitlik]].

When a service name recovered from stale conversation state is no longer valid for the organization (e.g., the service was removed), the agent clears it and triggers service clarification rather than returning a generic "no availability" error. This prevents the confusing "Bu hafta için uygun saatlerimiz yok" response when the real issue is a stale service, not actual unavailability.

For RESCHEDULE, when the LLM fails to extract `old_slot_date`/`old_slot_start` or incorrectly extracts the new desired time as the old time, the agent auto-populates them from the user's database bookings — picking the booking whose time differs from the new desired time when multiple bookings exist.

### Availability agent
Queries the database for available slots matching the requested service, resource, date, and time. Defined in [[agentic/agents/booking/availability_agent.py]].

When the requested date is beyond the organization's booking horizon (max available slot date), the agent **clamps** the date to `max_date` and proceeds with the query, rather than rejecting it. This ensures users always see the nearest available slots.

### Negotiation agent
When the requested slot is unavailable, proposes alternative nearby slots. Defined in [[agentic/agents/booking/negotiation_agent.py]].

### Booking agent

Performs the actual create/replace/cancel appointment mutation against the database. Defined in [[agentic/agents/booking/booking_agent.py]]. Handles PostgreSQL operations for booking CRUD.

Rescheduling requires explicit user confirmation. The booking_agent relies on `pending_reschedule=True` to execute, but upstream routing was resetting this flag to `False` on "evet" confirmations, causing an infinite "onaylıyor musunuz?" loop. Two fixes guard against this:
- `intent_agent` now preserves `pending_reschedule=True` when routing an affirmative reply to `booking_agent`.
- `booking_agent` also executes when it sees an explicit yes keyword (e.g. "evet") and the old/new slot context is present, even if the flag is missing.

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

**Stage 2 (completed): Read-path cleanup.** All booking agent read paths now use `state.get()` via [[agentic/booking/context_codec.py#get_booking_field]] (with legacy marker fallback for pre-migration sessions). Key changes:
- `booking_agent.py`: Replaced `_extract_booking_context_candidates` + `merge_booking_contexts` marker logic with direct `get_booking_field()` reads. Reconfirmation detection now compares `proposal_id` from state vs last booking_agent marker. Slot fields preserved in state return for pending confirmations (no longer cleared to None).
- `intent_classifier.py`: Replaced `iter_recent_booking_state_entries` marker scan with `get_booking_field()` for slot/service context. Clearing logic now nullifies state fields directly (no clearing-marker emission).
- `response_agent.py`: Deleted `_booking_state_passthrough()` — `take_last` reducers handle propagation automatically.
- `availability_agent.py`: `proposal_id` now written to `state_updates` (not just markers) so it propagates via GraphState.

**Stage 3 (completed): Write-path cleanup and state preservation.** Removed `build_booking_state_messages()` calls and fixed state preservation across turns. Key changes:
- `intent_classifier.py`: For booking intents, `service_name`/`service_id`/`resource_name`/`resource_id` are preserved from state instead of always cleared. When a booking confirmation is pending, `slot_date`/`slot_start`/`slot_end` are also preserved so time-only follow-ups (e.g. "9.30") reuse the prior date.
- `intent_agent.py`: Pending booking confirmation path now preserves `service_name`/`service_id`/`resource_name`/`resource_id` and carries `slot_date`/`slot_start` from state when the user sends a time-only or service-only update.
- `api/main.py`: Booking state fields persisted to and restored from `state_fields` in MongoDB via `_extract_booking_state_fields`/`_restore_booking_state_fields`.

## Booking state in GraphState

Booking-specific fields are defined directly in the shared [[agentic/core/state.py#GraphState]] (not a separate sub-state) to enable cross-subgraph persistence. Key fields:
- `pending_reschedule`, `pending_booking_confirmation`, `pending_cancel_confirmation` — Confirmation flow gates.
- `slot_date`, `slot_start`, `slot_end`, `slot_timezone`, `slot_date_end` — Current slot being worked on.
- `service_id`, `service_name`, `resource_id`, `resource_name` — Selected service/resource.
- `old_slot_date`, `old_slot_start`, `old_resource_id` — Previous slot for reschedule/cancel.
- `proposal_id`, `booking_status` — Booking epoch tracking and lifecycle status (promoted from BookingState in the Stage 1 schema promotion).
- `needs_clarification`, `pending_clarification_slot_start`, `pending_clarification_slot_date` — Split-turn clarification state (promoted).
- `available`, `availability_decision`, `is_range_query`, `is_flexible`, `time_preference` — Availability query context (promoted).
