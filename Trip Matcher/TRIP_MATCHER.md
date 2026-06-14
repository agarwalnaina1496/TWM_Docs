# Trip Matcher — Full Flow

## Overview

Trip Matcher takes a traveler from "I want to go somewhere" to "I know exactly where I'm going and why it's right for me" — without them needing to verify it anywhere else.

It is an independent, stateless feature. It does not know about Trip Planner. It does not communicate with Trip Planner. It reads Trip State at open, does its job, and writes back to Trip State when something changes.

Trip Matcher is built on two agents and a knowledge layer:

- **Scout** — Conversation Agent. The only interface the traveler sees.
- **Meridian** — Decision Engine. Runs all matching logic. Invisible to the traveler.
- **Destination KB** — Structured destination profiles. Meridian's data layer.

---

## Architecture

```
  Trip State
      ↕
    Scout  (conversation layer)
      ↕
   Meridian  (decision engine)
      ↕
Destination KB + Maps API
```

The traveler only ever talks to Scout. Meridian and the KB are completely invisible.

---

## Full Flow

### Stage 1: Intake

Scout opens with the current `matcher_inputs` from Trip State. If the user is starting fresh, all fields are null and Scout starts from scratch. If the user is returning, Scout resumes — it does not restart.

Scout begins collecting inputs through natural conversation. It starts with the highest-signal questions — trip goal, origin, travel month, budget — and builds from there. It does not ask for everything upfront. Inputs emerge through conversation.

Scout recognises passively mentioned details (group type, exclusions, preferences) and does not re-ask for things already said.

Trip State `stage` → `matching` as soon as any input is collected.

After each exchange, `trip_state.matcher_inputs` is updated with the latest inputs.

---

### Stage 2: Input Validation

Before calling Meridian, Scout checks:

- Are all 5 required inputs present? If not, ask for what is missing.
- Are any inputs ambiguous? Resolve before proceeding.
- Are any inputs in obvious conflict? Surface the conflict, ask the traveler to resolve it.

Scout never sends an incomplete or unresolved payload to Meridian.

---

### Stage 3: First Meridian Call

Scout sends a structured input payload to Meridian.

Meridian runs the full pipeline:
```
Step 0: Parse preferences and overrides
Step 1: Generate candidates from KB
Step 2: Hard constraint elimination
Step 3: Goal fit elimination
Step 4: Transport feasibility
Step 5: Travel time feasibility
Step 6: Geography validation
Step 7: Score and rank survivors
Step 8: Detect failure states
```

Meridian returns either a standard output (ranked shortlist + refinement hooks) or a failure output.

---

### Stage 4: Presentation

Scout translates Meridian's structured output into natural conversation.

- Lead with the top recommendation and the core reason it fits
- Show budget, reachability, weather, and tradeoffs
- Offer to show alternatives — do not dump all three at once
- Be honest about weaknesses

---

### Stage 5: Refinement Loop

Scout asks whether the recommendation lands. Based on the traveler's response and Meridian's refinement hooks, Scout asks targeted follow-up questions, updates the payload, and calls Meridian again.

`trip_state.matcher_inputs` is updated after each turn.

```
Call Meridian
    ↓
Present output
    ↓
Traveler responds
    ↓
Adjust: update inputs → call Meridian again
    OR
Confirm: write to Trip State → done
```

---

### Stage 6: Confirmation

When the traveler confirms a destination, Scout outputs the final result. Trip State is updated:

```json
{
  "stage": "matched",
  "destination": {
    "name": "Coorg",
    "confirmed_via": "matcher",
    "why_it_matched": "Scenic, peaceful, great for couples in October, easy from Bengaluru.",
    "shortlist_considered": ["Coorg", "Wayanad", "Munnar"]
  }
}
```

Trip Matcher's job is done. The user decides what to do next. They may move to Planner, they may not. That is not Trip Matcher's concern.

---

## Failure State Flow

When Meridian returns a failure state, Scout translates it into plain language and re-engages the traveler with a targeted question.

```
Meridian returns HARD_FAIL
    ↓
Scout: "Nothing matched all your criteria. The main conflict was [X]. Want to loosen that?"
    ↓
Traveler adjusts preference
    ↓
matcher_inputs updated → Meridian called again
```

See Failure State section in the Meridian doc for all failure types. See Scout doc for how each is communicated to the traveler.

---

## Success Metric

The traveler reaches a confident destination decision — grounded in their specific trip goal, travel style, and constraints — without needing to research or verify it anywhere else.
