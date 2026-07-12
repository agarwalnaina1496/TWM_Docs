# Stage Transitions

Stages are lifecycle state. They are not the same as Scout's per-turn `intent`.

## Forward Transitions

| From Stage | To Stage | Trigger |
|---|---|---|
| `new` | `matching` | Scout routes a turn to Matcher |
| `matching` | `matching` | Meridian asks a soft clarification with `NEEDS_CLARIFICATION` |
| `matching` | `recommended` | Meridian returns recommendation or matcher business output |
| `recommended` | `matched` | User selects or clearly confirms one destination/circuit |
| `matched` | `planning` | Scout routes a settled-destination turn to Planner |
| `planning` | `planned` | Planner completes a plan |

## Backward Transitions

| From Stage | To Stage | Trigger |
|---|---|---|
| `recommended` | `matching` | User rejects recommendations or asks to refine preferences |
| `matched` | `recommended` | User rejects selected destination but wants to choose from existing recommendations |
| `matched` | `matching` | User rejects selected destination and wants fresh recommendations |
| `planning` | `matched` | User changes itinerary details but keeps the same destination |
| `planning` | `recommended` | User changes destination from existing recommendations |
| `planning` | `matching` | User changes destination, budget, dates, duration, or trip goal significantly |

`recommendation_ready` is legacy and no longer part of the primary flow.
