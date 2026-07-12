# TripState

`TripState` is the source of truth for one trip conversation.

## Purpose

`TripState` carries what the traveler has actually told the system, plus lifecycle state needed by the product.

Scout reads the complete `TripState` on every turn and writes deltas. Meridian and Planner read the accumulated `trip_context` later.

## Design Principles

### Single Trip Scope

One `TripState` object represents one trip only.

Multiple trips for the same traveler must have separate `TripState` records.

### Progressive Discovery

`TripState` reflects only what is currently known.

Do not create placeholder fields for information that has not been discovered.

`trip_context` starts empty:

```json
"trip_context": {}
```

Scout adds fields only when the traveler supplies that signal. There are no predefined trip-context fields, no `required_inputs`, and no null placeholders.

### Raw Context First

`trip_context` stores raw extracted trip context in the traveler's wording verbatim wherever possible.

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
- advisor_state is owned by advice turns.
- matcher_state is owned by Meridian / Trip Matcher turns.
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

## trip_context

`trip_context` is an open object. It is not a form.

`trip_context` keeps common trip context directly at the top level and nests only phase-specific context:

```json
{
  "advisor": {},
  "matcher": {},
  "planner": {}
}
```

Use:

```text
top level -> common facts, constraints, preferences, travel history, timing, budget, companions
advisor -> direct advice/comparison/question Scout should answer
matcher -> matcher-related signals for deciding where to go
planner -> deferred itinerary/logistics/food/must-visit asks
```

Do not create empty phase buckets. All keys remain open-ended and traveler-wording-first.

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
  "preferences": {},
  "travel_history": {},
  "matcher": {
    "request": "Would love other suggestions from everyone"
  },
  "planner": {
    "requested_details": []
  }
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

`advisor_state` stores meaningful advice Scout has already given. It is not chat history.

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
      "created_at": "2026-07-09T00:00:00Z"
    }
  ]
}
```

`assistant_message` is stored verbatim, not summarized. Do not duplicate the user's message here; useful traveler-provided context belongs in `trip_context`.

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

`rejected_options` is optional legacy matcher state for recommendation options the traveler rejected during older refinement flows.

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
| Scout routes a turn to Matcher | `matching` |
| Meridian asks a soft clarification | `matching` |
| Meridian recommendation/failure output is returned to UI | `recommended` |
| User refines recommendations after `recommended` | `matching` |
| Destination or circuit confirmed | `matched` |
| User refines after `matched` | `matching` and selected destination is cleared by UI |
| Planner started | `planning` |
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
trip_context
advisor_state
```

Writes deltas for:

```text
trip_context
advisor_state
```

### Meridian

Reads:

```text
common trip_context fields
trip_context.matcher
trip_context.selected_option
matcher_state
```

Writes deltas for:

```text
trip_context
matcher_state.conversation_context
```

Returns recommendation output for UI to store in:

```text
matcher_state.recommendations
```

### UI

Owns deterministic lifecycle writes:

```text
stage
selected destination / option
matcher_state.recommendations
matcher_state.rejected_options
```

### Planner

Reads:

```text
trip_context
```

Writes:

```text
planner_state
stage
```
