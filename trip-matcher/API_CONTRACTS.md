# Trip Matcher API Contracts

This document records the UI/backend contract for Scout, Meridian, and shared `TripState`.

For the canonical state model, see [TripState](../TRIP_STATE.md).

## Endpoints

```text
POST /scout
POST /meridian
```

---

## POST /scout

Scout is the entry router and advice owner. The UI calls Scout while no specialist owns the active conversation.

Request:

```json
{
  "trip_state": {
    "stage": "matching",
    "trip_context": {},
    "advisor_state": {}
  },
  "message": "string | null"
}
```

`trip_state` is not the full app TripState. It must not include `trip_id`, `status`, `matcher_state`, `planner_state`, recommendations, or rejected options.

Response:

```json
{
  "message": "string",
  "state_delta": {
    "trip_context": {}
  },
  "intent": "advise | matcher | planner | null",
  "agent_meta": {
    "agent": "scout",
    "prompt_version": "string"
  }
}
```

Rules:

```text
- UI accepts only Scout-owned `state_delta.trip_context` fields and deep-merges them into TripState.
- Store every extracted traveler signal directly under trip_context using a specific, meaningful key.
- Preserve useful extracted values verbatim where possible, but do not copy the complete user query into context.
- Do not use generic catch-all keys such as request, question, or raw_message.
- Scout should not return required_inputs or a fixed preferences schema.
- Scout should not write stage, selected_option, recommendations, or rejected_options.
- Scout should not write recommendation_intent.
- Scout writes only new or updated traveler-provided fields under state_delta.trip_context.
- For `intent = advise`, UI stores the top-level Scout message in advisor_state and creates the advice artifact deterministically.
- intent = advise means Scout answered the current concern directly.
- intent = matcher means Meridian owns the visible reply.
- intent = planner means UI owns the current coming-soon reply; a future Planner will own itinerary output.
- intent = null means no Advise/Matcher/Planner phase was needed.
```

---

## POST /meridian

The UI calls Meridian automatically when Scout returns `intent = matcher`. It then routes later matching turns directly to Meridian while `active_agent = meridian`.

Request:

```json
{
  "trip_state": {
    "trip_context": {},
    "advisor_state": {
      "conversation_context": {
        "last_advisor_message": "string | null"
      }
    },
    "matcher_state": {
      "conversation_context": {
        "last_meridian_message": "string | null",
        "awaiting": "string | null"
      },
      "recommendations": [],
      "rejected_options": []
    }
  },
  "message": "string | null"
}
```

`trip_state` is Meridian's phase slice. It must not include `trip_id`, `status`, `stage`, `active_agent`, or `planner_state`.

The UI sends:

```text
trip_context
advisor_state.conversation_context.last_advisor_message
matcher_state
message, when a traveler turn is present
```

On initial handoff, UI first merges Scout's traveler-context delta and then builds this request. On continuation, `message` is the new clarification or refinement turn and Scout is not called again.

Response:

```json
{
  "message": "traveler-facing matcher reply",
  "state_delta": {
    "trip_context": {},
    "matcher_state": {
      "conversation_context": {
        "last_meridian_message": "same text as message",
        "awaiting": "string | null"
      }
    }
  },
  "status": "NEEDS_CLARIFICATION | SUCCESS | SOFT_FAIL | HARD_FAIL | BUDGET_FAIL | CONFLICT_FAIL",
  "generated_at": "ISO-8601 timestamp",
  "trip_type": "single | circuit | mixed | null",
  "agent_meta": {
    "agent": "meridian",
    "prompt_version": "string"
  },
  "options": [
    {
      "rank": 1,
      "type": "single | circuit",
      "name": "Destination or circuit name",
      "destination_id": "stable_destination_id_or_null",
      "circuit_id": "stable_circuit_id_or_null",
      "best_for": "who this option is best for / why this rank makes sense",
      "why_ranked_here": ["string"],
      "decision_summary": {
        "matches": ["string"],
        "tradeoffs": ["string"]
      },
      "sections": []
    }
  ]
}
```

`why_ranked_here` is required for every recommendation option. It should explain why this option has this rank by using material `trip_context` fields such as duration, travel month/season, budget, origin/reachability, companions, weather, crowd preference, and hard exclusions. Every useful field Meridian receives should be considered somewhere in ranking, matches, tradeoffs, sections, or option explanation.

Meridian should not return `match_sections`, `why_this_works_for_you`, `final_recommendation`, or `refinement_hooks` in the current contract.

Meridian may return `constraint_adjustment_suggestions` for `SOFT_FAIL`, `HARD_FAIL`, `BUDGET_FAIL`, or `CONFLICT_FAIL` when there are clear ways to adjust the ask. It is omitted for `SUCCESS`, `NEEDS_CLARIFICATION`, and whenever no useful adjustment exists; it is never `null`.

Top-level `version` and `relaxation_suggestions` are not part of the canonical response. Prompt provenance lives in `agent_meta.prompt_version`.

Meridian must not return:

```text
recommendation_intent
stage
MISSING_INPUTS
```

---

## UI Flow

```text
User sends message
  -> UI checks active_agent
  -> if Scout owns the turn, POST /scout with Scout phase slice
  -> UI deep-merges only Scout-owned trip_context delta
  -> UI derives routing from validated Scout intent
  -> if intent = advise, render Scout message
  -> if intent = matcher, UI sets active_agent = meridian and POSTs the deep-merged Meridian phase slice plus the same message
  -> if intent = planner, render UI-owned planner coming-soon message
  -> if intent = null, render Scout message when present
```

For matcher turns:

```text
Scout intent = matcher
  -> UI sets stage = matching when appropriate
  -> UI calls Meridian with current message
  -> UI deep-merges only Meridian-owned trip_context and matcher_state delta
  -> if status = NEEDS_CLARIFICATION, stage remains matching and active_agent remains meridian
  -> next traveler matching turn goes directly to Meridian
  -> otherwise UI clears active_agent, appends the terminal payload to matcher_state.recommendations, and sets stage = recommended
```

The backend does not merge deltas for the current UI-driven flow.

---

## Stage Rules

Stages are lifecycle, not turn intent:

```text
new
matching
recommended
matched
planning
planned
```

Current UI-owned transitions:

```text
Trip created -> new
Scout returns intent = matcher -> matching
Meridian returns NEEDS_CLARIFICATION -> matching
Meridian returns recommendation/failure output -> recommended
User chooses destination/circuit -> matched
User refines after matched -> matching and selected_option cleared
UI handles validated planner route or planning action -> planning
Plan complete in future -> planned
```

`recommendation_ready` is legacy and no longer part of the primary flow.

Advise is not a stage. Advice can happen while the trip is `new`, `matching`, `recommended`, `matched`, or `planning`.

---

## Confirmation

Card/button confirmation is deterministic:

```text
User chooses an option
  -> UI writes trip_context.selected_option
  -> UI sets stage = matched
```

Scout and Meridian should not write `selected_option`.

---

## Resume

On refresh, the UI reads `TripState` from localStorage.

```text
new + no trip_context -> show Scout welcome with active_agent = scout
new + trip_context -> show saved context and resume Scout-owned advice/entry
matching + active_agent = meridian -> restore Meridian context and route the next turn directly to Meridian
matching + active_agent = scout -> restore Scout-owned context
recommended -> show/review existing recommendations
matched -> show selected destination and planning CTA
planning -> show planning in progress / coming-soon flow
planned -> show plan-ready state
```

---

## Error Handling

Scout infrastructure failure:

```text
Do not merge the failed result.
Preserve valid TripState and the current owner.
Show retry action for the exact same turn without duplicating the traveler message.
```

Meridian infrastructure failure:

```text
Do not merge or append the failed result.
Keep `active_agent = meridian` and preserve valid matcher state.
Show retry action for the exact same turn without duplicating the traveler message.
```

Meridian business outputs such as `HARD_FAIL`, `SOFT_FAIL`, `BUDGET_FAIL`, and `CONFLICT_FAIL` are expected outputs and may be appended/rendered like other Meridian responses.

Starting a new journey clears pending retry state and resets `active_agent = scout`.

## Lifecycle Examples

### Clarification continuation

Scout returns `intent = matcher`. UI merges Scout's context delta, sets `active_agent = meridian`, and calls Meridian with the same traveler message. If Meridian returns `NEEDS_CLARIFICATION`, UI keeps Meridian active and sends the next traveler reply directly to Meridian with the persisted matcher context.

### Terminal outcome

Meridian returns any terminal business status. UI stores the response in recommendation history, clears `active_agent`, updates the lifecycle stage, and presents the available UI-owned action. The completed turn does not pass through Scout again.

### Retry

A retryable Scout or Meridian infrastructure failure does not mutate valid state. UI preserves the active owner and retries the exact failed turn without adding a duplicate traveler message.

### Refresh and resume

UI restores TripState from persistence. When `active_agent = meridian`, it restores matching context and routes the next traveler message directly to Meridian.

### New journey

UI clears pending retry and specialist context, creates a fresh TripState, and restores Scout as the entry owner.
