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
Helps the user find the right destination. Built on Scout (Conversation Agent) and Meridian (Decision Engine). Reads `matcher_inputs` from Trip State on open. Writes destination and updated inputs back on confirmation.

### Trip Planner (Phase 2)
Helps the user plan the trip once a destination is known. Reads destination and trip context from Trip State on open. Works even if Matcher was never used — user may have come in with a destination already in mind.

### Future Features
Any future feature follows the same pattern — reads relevant Trip State fields, does its job, writes back.

---

## Trip State

Trip State is the shared object all features read from and write to. Stored in localStorage during Phase 1. Migrates to database when user accounts are introduced — schema does not change, only the storage layer does.

### Schema

```json
{
  "version": "1.0",
  "created_at": "ISO timestamp",
  "updated_at": "ISO timestamp",

  "status": "free",
  "stage": "new",

  "matcher_inputs": {
    "budget": null,
    "budget_unit": "total",
    "duration_nights": null,
    "travel_month": null,
    "num_travelers": null,
    "origin_city": null,
    "trip_goal": null,
    "travel_style": [],
    "crowd_preference": null,
    "weather_preference": null,
    "group_type": null,
    "budget_flexibility": null,
    "exclusions": []
  },

  "destination": {
    "name": null,
    "confirmed_via": null,
    "why_it_matched": null,
    "shortlist_considered": []
  },

  "planner_context": {
    "accommodation_type": null,
    "specific_interests": [],
    "traveler_notes": null
  }
}
```

---

## Status and Stage

### Status

```
free        →  Exploratory. Everything is reversible.
committed   →  Payment made. Stage moves forward only.
```

Status starts as `free`. Moves to `committed` on payment. Never goes back.

### Stage

```
new         →  Trip State created. Nothing done yet.
matching    →  Matcher started. No destination confirmed.
matched     →  Destination confirmed (via Matcher or entered directly).
planning    →  Planner started. Plan not complete.
planned     →  Full trip plan complete.
booked      →  Trip booked. (Post-payment)
done        →  Trip completed. (Post-payment)
```

### Two Zones

**Free zone** — `new → matching → matched → planning → planned`

Stage is freely mutable. User can go back to Matcher from Planner, change destination, re-match, re-plan. Stage reflects *last known activity* — it moves backward if the user moves backward. No locks.

**Committed zone** — `planned → booked → done`

Triggered by payment. Stage moves forward only. User cannot regress to free zone activities.

### Stage Transitions

**Free zone (reversible)**

| Action | Stage becomes |
|---|---|
| Trip State created | `new` |
| Any Matcher input added | `matching` |
| Destination confirmed | `matched` |
| User returns to Matcher after matching | `matching` |
| Any Planner input added | `planning` |
| User returns to Matcher from Planner | `matching` |
| Plan complete | `planned` |

**Committed zone (forward only — triggered by payment)**

| Action | Stage becomes |
|---|---|
| Payment made | `booked` |
| Trip completed | `done` |

### Stage Notes

`matched` can be reached without ever going through `matching`. If the user enters a destination directly, stage goes `new → matched`. Stage reflects what has been done, not which feature was used.

`committed` is the only real gate in the system. Everything before it is exploratory.

Stage data in Trip State will power the dashboard in future — showing users where each trip stands at a glance.

---

## What Each Feature Reads and Writes

### Trip Matcher

**Reads:** `matcher_inputs` (to resume conversation, not restart it)

**Writes:**
- `matcher_inputs` — updated after each conversation turn
- `destination` — on confirmation
- `stage` — `matching` when started, `matched` on confirmation

### Trip Planner

**Reads:** `destination`, `matcher_inputs` (for budget, duration, group type context), `planner_context`

**Writes:**
- `planner_context` — updated as plan develops
- `stage` — `planning` when started, `planned` on completion

**Note:** Trip Planner works whether or not Matcher was used. If `destination.name` is null, Planner asks the user for it directly and writes it to Trip State.
