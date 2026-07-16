# TripState

`TripState` is the source of truth for one trip conversation.

## Purpose

`TripState` carries what the traveler has actually told the system, plus lifecycle state needed by the product.

The UI stores the complete `TripState` and sends each agent only its approved phase slice. Scout receives entry/advice turns. After matching handoff, Meridian receives later clarification and refinement turns directly until a terminal outcome.

## Design Principles

### Single Trip Scope

One `TripState` object represents one trip only.

Multiple trips for the same traveler must have separate `TripState` records.

### Progressive Discovery

`TripState` reflects only what is currently known.

Do not create placeholder fields for information that has not been discovered.

`trip_context` starts empty:

```json
{
  "trip_context": {}
}
```

Scout adds fields only when the traveler supplies that signal. There are no predefined trip-context fields, no `required_inputs`, and no null placeholders.

### Raw Context First

`trip_context` stores useful extracted trip signals in the traveler's wording verbatim wherever possible.

Verbatim preservation applies to each useful value, not to the complete user message, question, or request. Use specific keys that explain the reusable signal; do not use catch-all keys such as `request`, `question`, or `raw_message`.

Do not normalize, convert, score, infer, or compress at the TripState layer.

Values may be split into arrays/objects when the traveler lists multiple distinct items or when nesting preserves the relationship between signals.

Examples:

```json
{
  "travel_month": "September",
  "duration": "two week vacay",
  "budget": "Budget is not really a constraint so go crazy I suppose",
  "preferences": {
    "weather": "preferably somewhere with pleasant weather",
    "crowd_level": "not super crowded"
  }
}
```

Meridian and Planner may interpret this raw context later.

## Top-Level Shape

```json
{
  "trip_id": "trip_7f3a9c",
  "status": "free",
  "stage": "matching",
  "active_agent": "meridian",
  "trip_context": {},
  "advisor_state": {
    "conversation_context": {
      "last_advisor_message": null
    },
    "artifacts": []
  },
  "matcher_state": {
    "conversation_context": {
      "last_meridian_message": null,
      "awaiting": null
    },
    "recommendations": [],
    "rejected_options": []
  },
  "planner_state": null
}
```

## Key Rules

```text
- One TripState object represents one trip only.
- trip_context starts as {}.
- trip_context contains only discovered traveler-provided context.
- Do not create null placeholders in trip_context.
- Do not use required_inputs.
- Keep values verbatim wherever possible.
- active_agent is UI-owned routing state: scout, meridian, or null.
- advisor_state persistence is owned by the UI for advice turns.
- matcher_state conversation and rejection context is Meridian-owned; recommendation history is UI-owned.
- planner_state is null until planning starts.
```

## Lifecycle Fields

`status`:

```text
free      -> exploratory and reversible
committed -> payment made; stage moves forward only
```

`stage`:

```text
new
matching
recommended
matched
planning
planned
booked
done
```

For the current Scout/Meridian flow:

```text
new                  -> TripState created
matching             -> Matcher route active
recommended          -> Meridian recommendations have been generated and stored
matched              -> traveler confirmed a destination or circuit
planning             -> traveler moved into planning
```

`active_agent` is separate from `stage`:

```text
scout    -> UI sends the next entry/advice turn to Scout
meridian -> UI sends the next matching clarification/refinement directly to Meridian
null     -> no agent currently owns a specialist conversation
```

UI sets `active_agent = meridian` after a validated Scout matcher handoff, keeps it through `NEEDS_CLARIFICATION`, clears it on a terminal Meridian outcome, and resets it to `scout` for a new journey.

## trip_context

`trip_context` is an open object. It is not a form.

Every extracted signal is stored directly under `trip_context` using a specific, meaningful key:

```json
{
  "origin": "Gujarat",
  "budget": "around 70k (can also exceed if needed)",
  "destinations_considered": ["Mussoorie", "Rishikesh", "Haridwar"],
  "safety_concern": "female solo traveler"
}
```

Keys remain open-ended and traveler-wording-first. Arrays and nested objects are allowed when they preserve relationships between distinct signals, but the whole query must not be copied into context.

Example shape:

```json
{
  "trip_purpose": "this would be our second International trip together",
  "travelers": {
    "companion": "my husband"
  },
  "origin": "string",
  "travel_month": "September",
  "duration": "two week vacay",
  "budget": "Budget is not really a constraint so go crazy I suppose",
  "preferences": {
    "weather": "preferably somewhere with pleasant weather",
    "crowd_level": "not super crowded"
  },
  "planning_preference": "two cities and day trips from there"
}
```

These are examples, not required fields. Scout may create any sensible key when the traveler provides useful context.

### Example

```json
{
  "trip_purpose": "this would be our second International trip together",
  "travelers": {
    "companion": "my husband"
  },
  "travel_month": "September",
  "duration": "two week vacay",
  "budget": "Budget is not really a constraint so go crazy I suppose",
  "preferences": {
    "weather": "preferably somewhere with pleasant weather",
    "crowd_level": "not super crowded",
    "hotel_style": "We also don't want to keep changing our hotels every other day so planning two cities and day trips from there"
  },
  "travel_history": {
    "together": [
      "Austria (Vienna, Salzburg, Innsbruck, Hallstatt)",
      "Amsterdam",
      "Barcelona (got sick so couldn't properly visit it)",
      "Goa",
      "Corbett"
    ],
    "prior_to_marriage": [
      "Vietnam",
      "Thailand",
      "Dubai",
      "Kazakhstan",
      "Singapore",
      "Malaysia",
      "Greece",
      "Italy"
    ]
  },
  "current_plan_under_consideration": {
    "destination": "Italy (Rome and Puglia)",
    "concerns": [
      "I'm unsure what the weather would be like",
      "I am dreading the crowd in Rome"
    ]
  },
  "matcher": {
    "request": [
      "Would love other suggestions from everyone"
    ],
    "preferences": [
      "pleasant weather",
      "not super crowded",
      "two cities and day trips from there"
    ]
  }
}
```

## advisor_state

`advisor_state` stores meaningful advice Scout has already given. It is not chat history. For `intent = "advise"`, the UI derives this state deterministically from Scout's top-level `message`; Scout does not return a duplicate copy inside `state_delta`.

```json
{
  "conversation_context": {
    "last_advisor_message": "verbatim Scout advice"
  },
  "artifacts": [
    {
      "type": "advice",
      "source": "scout",
      "assistant_message": "verbatim Scout advice",
      "created_at": "2026-07-09T00:00:00Z",
      "agent_meta": {
        "agent": "scout",
        "prompt_version": "string"
      }
    }
  ]
}
```

`assistant_message` is stored verbatim from Scout's visible response, not summarized. The UI also preserves Scout agent metadata when available. Do not duplicate the user's message here; useful traveler-provided context belongs in `trip_context`.

## matcher_state

`matcher_state` is operational state for Meridian and destination matching.

```json
{
  "conversation_context": {
    "last_meridian_message": "Do you want this to stay within India, or are international options also open?",
    "awaiting": "domestic_or_international_scope"
  },
  "recommendations": [],
  "rejected_options": []
}
```

`conversation_context` carries only enough context for Meridian to continue a matcher clarification. It is not a conversation transcript.

Meridian may update `last_meridian_message` and `awaiting` when it asks one soft clarification.

`recommendations` stores Meridian output history. Each successful Meridian response, including business failures such as `HARD_FAIL` or `BUDGET_FAIL`, is appended to this array.

Infrastructure failures such as network errors or 5xx responses are not appended to `recommendations`.

`rejected_options` stores recommendation options the traveler has rejected during refinement. Meridian may return rejection-context updates in its owned delta; the UI validates and deep-merges them.

## planner_state

`planner_state` is `null` until planning starts.

Planner reads `trip_context` and writes planning-specific state later.

## Storage

Storage is an implementation detail:

```text
Today: localStorage
Later: database
```

The semantic contract should not change when storage moves from localStorage to a database.

## Stage Transition Reference

### Free Zone

| Action | Stage becomes |
|---|---|
| TripState created | `new` |
| UI handles Scout's validated Matcher handoff | `matching` |
| Meridian asks a soft clarification | `matching` |
| Meridian recommendation/failure output is returned to UI | `recommended` |
| User refines recommendations after `recommended` | `matching` |
| Destination or circuit confirmed | `matched` |
| User refines after `matched` | `matching` and selected destination is cleared by UI |
| UI enters the unavailable planning phase | `planning` |
| Plan complete | `planned` |

### Committed Zone

| Action | Stage becomes |
|---|---|
| Payment made | `booked` and `status` becomes `committed` |
| Trip completed | `done` |

## Feature Read/Write Ownership

### Scout

Reads:

```text
stage
trip_context
advisor_state
```

Writes deltas for:

```text
trip_context
```

Scout also returns the visible `message` and routing `intent`. It does not write `advisor_state`, lifecycle fields, or matcher/planner operational state.

### Meridian

Reads:

```text
trip_context
trip_context.selected_option
matcher_state
advisor_state.conversation_context.last_advisor_message
current traveler message
```

Writes deltas for:

```text
trip_context
matcher_state.conversation_context
matcher_state.rejected_options
```

Returns recommendation output for UI to store in:

```text
matcher_state.recommendations
```

### UI

Owns deterministic lifecycle writes:

```text
stage
active_agent
selected destination / option
matcher_state.recommendations
navigation and retry state
```

For Scout advice responses, UI also owns deterministic persistence of:

```text
advisor_state.conversation_context.last_advisor_message
advisor_state.artifacts
```

### Planner

Reads:

```text
trip_context
```

Writes:

```text
planner_state
```

The UI remains the sole owner of `stage` and `active_agent` when Planner is implemented.
