# TWM — System Architecture

## Design Philosophy

Trip Matcher and Trip Planner are independent, stateless features. They do not know about each other. They do not communicate with each other. Nothing orchestrates them. Nothing enforces a sequence.

The user decides how far they go. They can match and leave. They can plan without ever matching. They can go back and forth freely — until they pay. The sequencing exists in the user's mind and in the UI, not in the architecture.

What connects the features is shared state — Trip State. Each feature reads it at open, does its job, and writes back when something changes. The user experience feels like a connected journey because the state travels with them.

This is the same model as Swiggy. The menu doesn't know about the cart. The cart doesn't know about checkout. They all read and write the same shared state. The user navigates freely.

---

## Architecture

```
        Trip State
      (localStorage → DB)
       ↕       ↕       ↕
   Matcher  Planner  [Future features]
```

No orchestration layer. No routing logic. No entry points enforced by the system. Each feature is independently accessible. Each reads and writes Trip State.

---

## Features

### Trip Matcher (Phase 1)
Helps the user find the right destination. Built on Scout (Conversation Agent) and Meridian (Decision Engine). Reads `required_inputs` and `trip_preferences` from Trip State on open. Writes back as inputs are collected. Sets `selected_option` on confirmation.

### Trip Planner (Phase 2)
Helps the user plan the trip once a destination is known. Reads `selected_option`, `required_inputs`, and `trip_preferences` from Trip State on open. Works even if Matcher was never used — user may have come in with a destination already in mind.

### Future Features
Any future feature follows the same pattern — reads relevant Trip State fields, does its job, writes back.

---

## Trip State

Trip State is the shared object all features read from and write to. Stored in localStorage during Phase 1. Migrates to database when user accounts are introduced — schema does not change, only the storage layer does.

See `TRIP_STATE.md` for full field definitions, design principles, and stage transition reference.

### Schema

```json
{
  "trip_id": "trip_7f3a9c",
  "status": "free",
  "stage": "matching",

  "trip_context": {
    "required_inputs": {
      "origin_city": null,
      "budget": null,
      "budget_unit": null,
      "duration_nights": null,
      "num_travelers": null,
      "travel_month": null
    },
    "preferences": {},
    "selected_option": null
  },

  "matcher_state": {
    "generate_ready": false,
    "conversation_context": {
      "last_scout_message": null,
      "awaiting": null
    },
    "last_recommendations": null
  },

  "planner_state": null
}
```

### Key Field Notes

`trip_context` — shared across Matcher and Planner. Contains required inputs, all trip preferences, and the selected option. Both features read from this. Neither feature has visibility into the other's state object.

`trip_context.required_inputs` — six pre-defined nullable fields. Only pre-defined nulls in the schema.

`trip_context.preferences` — dynamic. Only discovered information. No null fields. Scalar preferences include a `confidence` score (`explicit | high | medium | low`). This is why `trip_goal` is not in `required_inputs` — it belongs to preferences with a confidence rating.

`trip_context.selected_option` — set when traveler confirms. `{ type: "destination" | "circuit", id: "..." }`. Lives in `trip_context` not `matcher_state` because it is Matcher's output that Planner consumes.

`matcher_state` — Matcher-only. Planner never reads or writes this.

`matcher_state.generate_ready` — UI shows Generate button when `true`. User always pulls the trigger.

`matcher_state.conversation_context` — carries just enough for Scout to resume without history. `last_scout_message` and `awaiting`. Not a summary — everything meaningful is already in `trip_context`.

`matcher_state.last_recommendations` — full Meridian output. Survives page refresh so Scout can re-engage with what was shown.

`planner_state` — null until destination confirmed. Has its own `conversation_context` and `preferences` for planning-specific signals.

`traveler_profile` — not in TripState. Managed separately in Phase 3+ when accounts are introduced. Backend combines TripState + TravelerProfile before calling agents — TripState never needs to know about it.

---

## Status and Stage

### Status

```
free        →  Exploratory. Everything is reversible.
committed   →  Payment made. Stage moves forward only.
```

### Stage

**Free zone (reversible):**
```
new        →  Trip State created. Nothing done yet.
matching   →  Scout collecting inputs.
ready      →  Scout signalled ready. Generate button visible.
matched    →  Destination or circuit confirmed.
planning   →  Planner started. Plan not complete.
planned    →  Full trip plan complete.
```

**Committed zone (forward only — triggered by payment):**
```
booked     →  Trip booked.
done       →  Trip completed.
```

### Stage Transitions

**Free zone (reversible)**

| Action | Stage becomes |
|---|---|
| Trip State created | `new` |
| Scout starts collecting | `matching` |
| Scout signals ready | `ready` |
| User sends message after `ready` | `matching` |
| Destination confirmed | `matched` |
| User returns to Matcher from Planner | `matching` |
| Planner started | `planning` |
| Plan complete | `planned` |

**Committed zone (forward only)**

| Action | Stage becomes |
|---|---|
| Payment made | `booked` + `status → committed` |
| Trip completed | `done` |

---

## What Each Feature Reads and Writes

### Trip Matcher

**Reads:** `trip_context.required_inputs`, `trip_context.preferences`, `matcher_state`

**Writes:**
- `trip_context.required_inputs` — as Scout collects inputs
- `trip_context.preferences` — as Scout discovers preferences
- `trip_context.selected_option` — on confirmation
- `matcher_state.generate_ready` — when Scout signals ready
- `matcher_state.conversation_context` — after every turn
- `matcher_state.last_recommendations` — after Meridian runs
- `stage` — updated through matching lifecycle

### Trip Planner

**Reads:** `trip_context` (all fields — destination, required inputs, preferences)

**Writes:**
- `planner_state` — conversation context, planning preferences, itinerary
- `stage` — `planning` when started, `planned` on completion

**Does not read or write:** `matcher_state`
- `stage` — `planning` when started, `planned` on completion

**Note:** Trip Planner works whether or not Matcher was used. If `destination.name` is null, Planner asks the user for it directly and writes it to Trip State.
