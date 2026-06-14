# Trip Matcher — Full Flow

## Overview

Trip Matcher is Phase 1 of TravelWithMe. Its job is to take a traveler from "I want to go somewhere" to "I know exactly where I'm going and why it's right for me" — without them needing to verify it anywhere else.

It is built on two agents and a knowledge layer:

- **Scout** — Conversation Agent. The only interface the traveler sees.
- **Meridian** — Decision Engine. Runs all matching logic. Invisible to the traveler.
- **Destination KB** — Structured destination profiles. Meridian's data layer.

---

## Architecture

```
Traveler
    ↕  (conversation)
  Scout
    ↕  (structured payload)
 Meridian
    ↕  (semantic query + structured filters)
Destination KB + Maps API
```

The traveler only ever talks to Scout. Meridian and the KB are completely invisible. The traveler's experience is a conversation. The intelligence behind it is Meridian.

---

## Full Flow

### Stage 1: Intake

Scout opens the conversation and begins collecting inputs. It starts with the highest-signal questions — trip goal, origin, travel month, budget — and builds naturally from there.

Scout does not ask for everything upfront. Inputs emerge through the conversation. Scout recognizes passively mentioned details (group type, exclusions, preferences) and doesn't re-ask for things already said.

---

### Stage 2: Input Validation

Before calling Meridian, Scout checks:

- Are all 5 required inputs present? If not, ask for what's missing.
- Are any inputs ambiguous? If so, resolve before proceeding.
- Are any inputs in obvious conflict? If so, surface the conflict and ask the traveler to resolve it.

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
- Offer to show alternatives — don't dump all three at once
- Be honest about weaknesses

---

### Stage 5: Refinement Loop

Scout asks whether the recommendation lands. Based on the traveler's response and Meridian's refinement hooks, Scout asks targeted follow-up questions, updates the payload, and calls Meridian again.

The loop continues until the traveler confirms a destination or exits.

```
Call Meridian
    ↓
Present output
    ↓
Traveler responds
    ↓
Adjust: update payload → call Meridian again
    OR
Confirm: capture decision → exit
```

---

### Stage 6: Decision Confirmation

When the traveler signals confidence in a destination, Scout confirms the decision and captures the full context as the Trip Matcher handoff artifact.

This artifact becomes the input to Phase 2 (Trip Planner).

---

## Failure State Flow

When Meridian returns a failure state, Scout does not dead-end. It translates the failure into a plain-language explanation and re-engages the traveler with a targeted question.

```
Meridian returns HARD_FAIL
    ↓
Scout: "Nothing matched all your criteria. The main conflict was [X]. Want to loosen that?"
    ↓
Traveler adjusts preference
    ↓
Updated payload → Meridian again
```

See the Failure State section in the Meridian doc for all failure types and the Scout doc for how each is communicated.

---

## Handoff Artifact (Trip Matcher → Trip Planner)

When Trip Matcher is complete, Scout outputs a structured artifact capturing the full decision context. This is the input to Phase 2.

```json
{
  "confirmed_destination": "Coorg",
  "travel_month": "October",
  "duration_nights": 3,
  "num_travelers": 2,
  "group_type": "couple",
  "trip_goal": "relaxation",
  "travel_style": ["scenic", "relaxed"],
  "budget_total": 45000,
  "budget_flexibility": "moderate",
  "origin_city": "Bengaluru",
  "exclusions": ["no_trekking"],
  "why_it_matched": "Scenic hill destination well-suited for couples, peaceful atmosphere, accessible from Bengaluru, great in October.",
  "traveler_notes": "Prefers a homestay over a resort. Not interested in organised tours."
}
```

---

## Success Metric

The traveler reaches a confident destination decision — grounded in their specific trip goal, travel style, and constraints — without needing to research or verify it anywhere else.
