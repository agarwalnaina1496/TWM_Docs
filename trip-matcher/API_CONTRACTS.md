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

Scout is the single conversational entry point. The UI sends Scout's phase slice and the latest user message.

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
  "intent": "advise | matcher | planner | null"
}
```

Rules:

```text
- UI deep-merges state_delta into TripState.
- trip_context keeps common trip context directly at the top level.
- Put matcher-related signals under trip_context.matcher.
- Put planner-related signals under trip_context.planner.
- Put advice-only signals under trip_context.advisor.
- Do not create empty phase buckets.
- Scout should not return required_inputs or a fixed preferences schema.
- Scout should not write stage, selected_option, recommendations, or rejected_options.
- Scout should not write recommendation_intent.
- For `intent = advise`, Scout may write `advisor_state` with verbatim advice artifacts.
- For any other intent, Scout should write only `trip_context`; phase-owned state stays empty.
- intent = advise means Scout answered the current concern directly.
- intent = matcher means Meridian owns the visible reply.
- intent = planner means Planner owns the visible reply.
- intent = null means no Advise/Matcher/Planner phase was needed.
```

---

## POST /meridian

The UI calls Meridian automatically when Scout returns `intent = matcher`.

Request:

```json
{
  "trip_state": {
    "trip_context": {
      "...common trip context fields": "...",
      "matcher": {}
    },
    "matcher_state": {
      "conversation_context": {
        "last_meridian_message": "string | null",
        "awaiting": "string | null"
      },
      "recommendations": [],
      "rejected_options": []
    }
  }
}
```

`trip_state` is Meridian's phase slice. It must not include `trip_id`, `status`, `stage`, `advisor_state`, `planner_state`, or the raw user message.

The UI sends only:

```text
common trip_context fields
trip_context.matcher
trip_context.selected_option, when present
matcher_state
```

The UI must exclude `trip_context.planner` and `trip_context.advisor` from the normal Meridian request.

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
  "version": "matcher_v2",
  "trip_type": "single | circuit | mixed | null",
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

`why_ranked_here` is required for every recommendation option. It should explain why this option has this rank by using material `trip_context.matcher` fields plus fit-affecting common trip context such as duration, travel month/season, budget, origin/reachability, companions, weather, crowd preference, and hard exclusions. Every useful field Meridian receives should be considered somewhere in ranking, matches, tradeoffs, sections, or option explanation.

Meridian should not return `match_sections`, `why_this_works_for_you`, `final_recommendation`, or `refinement_hooks` in the current contract.

Meridian may return `relaxation_suggestions` for failure states when there are clear ways to adjust the ask. The core contract should stay limited to fields the UI/orchestration uses.

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
  -> POST /scout with Scout phase slice
  -> UI deep-merges Scout state_delta
  -> UI applies stage from Scout intent
  -> if intent = advise, render Scout message
  -> if intent = matcher, POST /meridian with Meridian phase slice in the same chat turn
  -> if intent = planner, render UI-owned planner coming-soon message
  -> if intent = null, render Scout message when present
```

For matcher turns:

```text
Scout intent = matcher
  -> UI sets stage = matching when appropriate
  -> UI calls Meridian
  -> UI deep-merges Meridian state_delta
  -> if Meridian status = NEEDS_CLARIFICATION, stage remains matching
  -> otherwise UI appends the Meridian payload to matcher_state.recommendations
  -> UI sets stage = recommended
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
Scout returns intent = planner -> planning
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
new + no trip_context -> show Scout welcome
new + trip_context -> show saved context and resume chat
matching -> show context and continue conversation
recommended -> show/review existing recommendations
matched -> show selected destination and planning CTA
planning -> show planning in progress / coming-soon flow
planned -> show plan-ready state
```

---

## Error Handling

Scout infrastructure failure:

```text
Do not update localStorage.
Show retry message.
User can retry with the same tripState.
```

Meridian infrastructure failure:

```text
Do not append to recommendations.
Show retry state.
User can retry with the same tripState.
```

Meridian business outputs such as `HARD_FAIL`, `SOFT_FAIL`, `BUDGET_FAIL`, and `CONFLICT_FAIL` are expected outputs and may be appended/rendered like other Meridian responses.
