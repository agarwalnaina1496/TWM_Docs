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
- Preserve each useful extracted value verbatim under a semantic key, but do not copy the complete user query into context.
- Preserve distinctions and qualifiers that materially affect advice or matching; do not infer adjacent traveler facts.
- Do not use generic catch-all keys such as request, question, or raw_message.
- Scout should not return required_inputs or a fixed preferences schema.
- Scout should not write stage, selected_option, recommendations, or rejected_options.
- Scout should not write recommendation_intent.
- Scout writes only new or updated traveler-provided fields under state_delta.trip_context.
- For `intent = advise`, UI stores the top-level Scout message in advisor_state and creates the advice artifact deterministically.
- intent = advise means Scout answered the current general-information concern directly and did not perform recommendation work.
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

Meridian first addresses the current matching ask using known context, then evaluates readiness for the requested recommendation rather than enforcing a universal required-field list. When one material clarification is required, it gives brief useful guidance and asks exactly one targeted question; after the answer, it recommends when ready. Constraint accounting, explicit assumptions, traveler-specific trade-offs, and circuit feasibility are canonical in [Meridian](MERIDIAN.md).

Response:

```json
{
  "message": "Here are the strongest destination matches for what matters to you.",
  "state_delta": {
    "trip_context": {},
    "matcher_state": {
      "conversation_context": {
        "last_meridian_message": "Here are the strongest destination matches for what matters to you.",
        "awaiting": null
      }
    }
  },
  "status": "SUCCESS",
  "generated_at": "2026-07-17T12:00:00Z",
  "trip_type": "single",
  "agent_meta": {
    "agent": "meridian",
    "prompt_version": "string"
  },
  "traveler_criteria": [
    {
      "id": "comfortable_weather",
      "label": "Comfortable weather for the travel month",
      "requirement_type": "PREFERENCE",
      "source_context_paths": ["travel_month", "weather_preference"]
    }
  ],
  "options": [
    {
      "rank": 1,
      "type": "single",
      "name": "Destination name",
      "destination_id": "stable_destination_id",
      "circuit_id": null,
      "summary": "Concise option-level orientation",
      "evaluations": [
        {
          "criterion_id": "comfortable_weather",
          "outcome": "TRADEOFF",
          "conclusion": "The month is scenic but rainfall can affect outdoor flexibility.",
          "details": [
            {
              "type": "facts",
              "facts": [
                { "label": "Season pattern", "value": "Rain is common in this period." }
              ]
            }
          ],
          "tradeoffs": ["Keep outdoor plans flexible and verify the forecast near departure."]
        }
      ],
      "other_considerations": ["Weekend traffic can increase transfer time."]
    }
  ]
}
```

`traveler_criteria` defines the material ask once at response level. Each criterion has a unique `id`, a unique traveler-facing `label`, its `HARD` or `PREFERENCE` meaning, and the TripContext paths that supplied it. A source path belongs to only one criterion. Every option must evaluate every criterion exactly once through `criterion_id`; unknown, duplicate, or missing references make the recommendation payload invalid.

The top-level `message` introduces the ranking. `summary` orients the traveler to one option. Each evaluation `conclusion` answers one criterion, `details` supplies approved `bullets`, `facts`, or `cost_breakdown` evidence, and `tradeoffs` stays attached to the affected criterion. `other_considerations` is only for residual information that does not belong to one criterion.

`MATCH` forbids trade-offs. `TRADEOFF` and `MISMATCH` require them, and a `HARD` criterion cannot be a `MISMATCH`. Cost values use inclusive `minimum`/`maximum` ranges in one ISO currency. Missing numeric costs and totals are omitted rather than encoded or rendered as zero.

The canonical payload has no option verdict, note block, fixed report sections, or itinerary. Superseded fields `best_for`, `why_ranked_here`, `decision_summary`, and `sections` are removed.

Approved evaluation detail shapes:

```json
[
  {
    "type": "bullets",
    "items": ["One concise supporting point"]
  },
  {
    "type": "facts",
    "facts": [
      { "label": "Typical character", "value": "Misty and rain-prone in this period" }
    ]
  },
  {
    "type": "cost_breakdown",
    "currency": "INR",
    "items": [
      {
        "label": "Stay",
        "per_person": { "minimum": 5000, "maximum": 7000 },
        "group": { "minimum": 10000, "maximum": 14000 }
      }
    ],
    "per_person_total": { "minimum": 5000, "maximum": 7000 },
    "group_total": { "minimum": 10000, "maximum": 14000 }
  }
]
```

A range requires finite non-negative `minimum` and `maximum` values with `maximum >= minimum`. When both group and per-person estimates are present, the group range cannot be lower than the per-person range.

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
  -> UI validates the business status
  -> if status = NEEDS_CLARIFICATION, UI deep-merges the owned delta, keeps stage = matching and active_agent = meridian, and routes the next matching turn directly to Meridian
  -> if a terminal response has options, UI validates traveler criteria and every option evaluation reference before any state mutation
  -> for a valid terminal response, UI deep-merges the owned delta, clears active_agent, appends the payload to matcher_state.recommendations, and sets stage = recommended
```

An invalid terminal recommendation contract is treated as an integration failure: the UI preserves the last valid TripState and saved recommendation history. Expected business failures without viable options remain valid terminal outcomes.

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
