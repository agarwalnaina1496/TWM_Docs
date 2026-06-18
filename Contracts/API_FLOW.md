# Trip Matcher — Technical API Flow

## Overview

The UI is the only orchestrator. It calls Scout and Meridian through separate backend endpoints. Scout and Meridian do not know about each other. The backend is stateless — every request includes full context.

TripState is the single source of truth. It lives in localStorage. No conversation array is maintained anywhere. Scout operates entirely on TripState — reading it on every call, writing back via `state_delta` after every turn. Page refresh is fully survivable.

The user controls when recommendations are generated. Scout signals readiness. The user pulls the trigger.

---

## Backend Architecture

```
UI (Browser)
    ↕
Backend (single service — Node.js / Python)
    ├── POST /api/scout/message    →  Anthropic API (Scout system prompt)
    ├── POST /api/scout/present    →  Anthropic API (Scout system prompt)
    └── POST /api/meridian/match   →  Anthropic API (Meridian system prompt)
                                       + Supabase (KB query)
```

No Anthropic API keys exposed to the client. All LLM calls are backend-side.

---

## API Endpoints

---

### POST /api/scout/message

Called on every user message during the matching conversation.
Scout reads TripState, extracts inputs, and returns a reply and a state delta.

#### Request

```json
{
  "trip_state": {
    "trip_id": "trip_7f3a9c",
    "status": "free",
    "stage": "matching",
    "trip_context": {
      "required_inputs": {
        "origin_city": "Bengaluru",
        "budget": null,
        "budget_unit": null,
        "duration_nights": null,
        "num_travelers": 3,
        "travel_month": "September"
      },
      "preferences": {
        "trip_goal": { "value": "Bachelorette", "confidence": "explicit" },
        "group_type": "friends"
      },
      "selected_option": null
    },
    "matcher_state": {
      "generate_ready": false,
      "conversation_context": {
        "last_scout_message": "What's your total budget for the trip?",
        "awaiting": "budget"
      },
      "last_recommendations": null
    },
    "planner_state": null
  },
  "message": "Around 30k total, 3 nights"
}
```

| Field | Required | Description |
|---|---|---|
| `trip_state` | Yes | Full current TripState from localStorage. Scout reads this instead of conversation history. |
| `message` | Yes | The user's latest message. |

No `conversation` array. No session ID. Every request is self-contained.

#### Response

```json
{
  "message": "Got it — ₹30,000 total for 3 nights. Is there anything else you'd like me to factor in? Travel style, crowd preferences, anything specific you're hoping for?",
  "state_delta": {
    "trip_context": {
      "required_inputs": {
        "budget": 30000,
        "budget_unit": "total",
        "duration_nights": 3
      },
      "preferences": {}
    },
    "matcher_state": {
      "generate_ready": false,
      "conversation_context": {
        "last_scout_message": "Got it — ₹30,000 total for 3 nights. Is there anything else you'd like me to factor in?",
        "awaiting": "additional_preferences"
      }
    }
  },
  "stage": "matching",
  "generate_ready": false,
  "confirmed_destination": null
}
```

After user says "nothing else" and Scout signals ready:

```json
{
  "message": "Perfect — I have everything I need. Ready to find recommendations when you are.",
  "state_delta": {
    "trip_context": {
      "required_inputs": {},
      "preferences": {
        "crowd_tolerance": { "value": "low", "confidence": "high" }
      }
    },
    "matcher_state": {
      "generate_ready": true,
      "conversation_context": {
        "last_scout_message": "Perfect — I have everything I need. Ready to find recommendations when you are.",
        "awaiting": null
      }
    }
  },
  "stage": "ready",
  "generate_ready": true,
  "confirmed_destination": null
}
```

| Field | Always present | Description |
|---|---|---|
| `message` | Yes | Scout's reply — render in chat UI. |
| `state_delta` | Yes | Only fields updated this turn. Split into `trip_context` and `matcher_state` sections. UI deep-merges each section into TripState and writes to localStorage. Empty objects `{}` if nothing changed in that section. |
| `stage` | Yes | Updated stage. UI writes to TripState. |
| `generate_ready` | Yes | `true` when Scout has collected required inputs and asked "anything else". UI shows Generate button. |
| `confirmed_destination` | Yes | `null` during collection. Populated when user confirms a destination. |

**Merging `state_delta`:**

UI deep-merges `state_delta.trip_context.required_inputs` into `trip_state.trip_context.required_inputs`.
UI deep-merges `state_delta.trip_context.preferences` into `trip_state.trip_context.preferences`.
UI replaces `trip_state.matcher_state` fields from `state_delta.matcher_state`.

Only include changed fields in `state_delta` — never send the full object back. UI merges, not replaces.

---

### POST /api/meridian/match

Called by the UI when the user taps "Generate Recommendations".
Never called automatically. Never called by Scout.

The backend extracts `required_inputs` and `preferences` from `trip_context` before passing to Meridian.

#### Request

```json
{
  "trip_context": {
    "required_inputs": {
      "origin_city": "Bengaluru",
      "budget": 30000,
      "budget_unit": "total",
      "duration_nights": 3,
      "num_travelers": 3,
      "travel_month": "September"
    },
    "preferences": {
      "trip_goal": { "value": "Bachelorette", "confidence": "explicit" },
      "crowd_tolerance": { "value": "low", "confidence": "high" },
      "group_type": "friends",
      "travel_style": ["relaxed", "social"],
      "nuanced_preferences": [
        "Wants a chill celebration vibe, not a big party scene",
        "Prefers boutique or aesthetic stays"
      ]
    }
  }
}
```

UI sends `trip_state.trip_context`. Backend handles the rest.

| Field | Required | Description |
|---|---|---|
| `trip_context` | Yes | From TripState. Backend merges `required_inputs` and `preferences` before building Meridian's prompt context. |

#### Response (SUCCESS)

```json
{
  "status": "SUCCESS",
  "trip_type": "single",
  "budget_basis": {
    "total": 30000,
    "per_person": 10000,
    "num_travelers": 3
  },
  "options": [
    {
      "rank": 1,
      "type": "single",
      "name": "Pondicherry",
      "destination_id": "pondicherry_puducherry",
      "match_sections": [
        {
          "type": "budget",
          "heading": "Fits your ₹30,000 budget",
          "stay": {
            "per_night_per_person": 2000,
            "nights": 2,
            "total": 6000,
            "note": "Boutique guesthouses in White Town"
          },
          "food": { "per_day_per_person": 600, "days": 3, "total": 5400 },
          "travel": {
            "per_person": 2000,
            "total": 6000,
            "description": "Bus or train round trip from Bengaluru"
          },
          "activities": {
            "per_person": 800,
            "total": 2400,
            "description": "Café hopping, beach time, one spa session"
          },
          "per_person_total": 9800,
          "group_total": 29400,
          "budget_given": 30000,
          "verdict": "Just within budget without cutting on comfort"
        },
        {
          "type": "trip_goal",
          "heading": "Works well for a bachelorette-style friend trip",
          "points": [
            "Boutique stays and aesthetic cafés",
            "Beachside brunches and rooftop sunset spots",
            "Easygoing bar scene — celebratory without overwhelming"
          ]
        },
        {
          "type": "crowd_preference",
          "heading": "Quieter than usual in September",
          "points": [
            "September is shoulder season — tourist volume drops",
            "Better availability across stays and restaurants"
          ]
        },
        {
          "type": "weather",
          "heading": "Some rain expected — plan flexibly",
          "contextual": true,
          "points": [
            "Warm and humid with intermittent showers",
            "Rain usually short bursts — beach time still possible"
          ]
        }
      ],
      "tradeoffs": [
        { "point": "September rain may interrupt beach plans", "affects": "trip_goal" },
        { "point": "Nightlife is low-key — no large clubs", "affects": "trip_goal" }
      ]
    }
  ],
  "final_recommendation": {
    "best_match": "Pondicherry",
    "best_match_reason": "Best balance of bachelorette vibe, manageable travel, shoulder-season quiet",
    "alternative_1": "Coorg",
    "alternative_1_reason": "Shorter travel, more intimate stay-based celebration",
    "alternative_2": "Gokarna",
    "alternative_2_reason": "Most affordable, least crowded"
  },
  "refinement_hooks": {
    "weakest_scoring_factor": "Seasonality — all three carry rain risk in September",
    "constraint_with_highest_elimination": "Bachelorette goal eliminated high-crowd party destinations",
    "budget_headroom": "comfortable",
    "seasonality_note": "September is monsoon or shoulder for all three options",
    "nuanced_preference_gaps": null
  }
}
```

#### Response (FAILURE)

```json
{
  "status": "HARD_FAIL",
  "message": "No destinations matched all the stated constraints",
  "eliminating_constraints": ["crowd_tolerance: low", "travel_month: September"],
  "relaxation_suggestions": [
    "Consider October — crowds similar but rain significantly lower",
    "Moderate crowd tolerance slightly"
  ],
  "surviving_destinations": []
}
```

---

### POST /api/scout/present

Called by the UI immediately after `/api/meridian/match` returns — SUCCESS or FAILURE.
Scout translates Meridian's output into a natural conversational message.

#### Request

```json
{
  "trip_state": {
    "trip_id": "trip_7f3a9c",
    "status": "free",
    "stage": "ready",
    "trip_context": {
      "required_inputs": {
        "origin_city": "Bengaluru",
        "budget": 30000,
        "budget_unit": "total",
        "duration_nights": 3,
        "num_travelers": 3,
        "travel_month": "September"
      },
      "preferences": {
        "trip_goal": { "value": "Bachelorette", "confidence": "explicit" },
        "crowd_tolerance": { "value": "low", "confidence": "high" },
        "group_type": "friends",
        "travel_style": ["relaxed", "social"]
      },
      "selected_option": null
    },
    "matcher_state": {
      "generate_ready": true,
      "conversation_context": {
        "last_scout_message": "Perfect — I have everything I need. Ready to find recommendations when you are.",
        "awaiting": null
      },
      "last_recommendations": null
    },
    "planner_state": null
  },
  "meridian_output": { "status": "SUCCESS", "options": ["..."] },
  "context": "first_presentation"
}
```

| Field | Required | Description |
|---|---|---|
| `trip_state` | Yes | Full current TripState. Scout uses this to frame the presentation. |
| `meridian_output` | Yes | Full Meridian response — SUCCESS or FAILURE. |
| `context` | Yes | `first_presentation` — first time showing recommendations. `refinement` — after user adjusted inputs. `post_confirmation` — destination confirmed, Scout wraps up. |

#### Response

```json
{
  "message": "Found three options that work. Pondicherry comes up top — fits your ₹30,000 budget cleanly at about ₹9,800 per person, good bachelorette vibe with boutique cafés and a relaxed beach scene, and September keeps the crowds down. Rain is something to plan around but usually short bursts. Want me to walk through why it ranks first, or see all three?",
  "state_delta": {
    "trip_context": {
      "required_inputs": {},
      "preferences": {}
    },
    "matcher_state": {
      "generate_ready": false,
      "conversation_context": {
        "last_scout_message": "Found three options that work. Pondicherry comes up top...",
        "awaiting": "user_response_to_recommendations"
      },
      "last_recommendations": {
        "options": ["pondicherry_puducherry", "coorg_karnataka", "gokarna_karnataka"],
        "best_match": "pondicherry_puducherry",
        "generated_at": "2025-09-15T10:45:00Z",
        "meridian_output": {}
      }
    }
  },
  "stage": "ready",
  "generate_ready": false,
  "confirmed_destination": null
}
```

---

## Full UI Flow

### Stage 1 — Input Collection

```
User sends message
        ↓
POST /api/scout/message
  body: { trip_state, message }
        ↓
Response received
        ↓
UI does three things in order:

  1. Deep-merge state_delta into trip_state
     Write updated trip_state to localStorage

  2. Update trip_state.stage and
     trip_state.matcher_state.generate_ready
     Write to localStorage

  3. Render response.message in chat UI
```

```
if generate_ready = false:
    Hide Generate button if visible
    Wait for next message
    → Loop

if generate_ready = true:
    Show "Generate Recommendations" button
    Wait for user to tap
```

---

### Stage 2 — User Taps Generate

```
User taps "Generate Recommendations"
        ↓
Button enters loading state
        ↓
POST /api/meridian/match
  body: { trip_context: trip_state.trip_context }
        ↓
Meridian response received
        ↓
POST /api/scout/present
  body: {
    trip_state,
    meridian_output,
    context: "first_presentation"
  }
        ↓
Response received
        ↓
UI deep-merges state_delta into trip_state → localStorage
Render Scout's message in chat
Hide Generate button
```

---

### Stage 3 — Refinement

```
User sends adjustment message
        ↓
POST /api/scout/message
  body: { trip_state, message }
        ↓
Scout extracts adjustment → returns state_delta
UI merges into trip_state → localStorage
        ↓
if generate_ready: true:
    Show Generate button
    User taps
        ↓
POST /api/meridian/match (updated trip_context)
POST /api/scout/present (context: "refinement")
        ↓
Updated recommendations shown
```

---

### Stage 4 — Confirmation

```
User confirms destination
        ↓
POST /api/scout/message
  body: { trip_state, message }
        ↓
Scout returns:
{
  "message": "Pondicherry it is.",
  "state_delta": {
    "trip_context": {
      "required_inputs": {},
      "preferences": {},
      "selected_option": {
        "type": "destination",
        "id": "pondicherry_puducherry"
      }
    },
    "matcher_state": {
      "generate_ready": false,
      "conversation_context": {
        "last_scout_message": "Pondicherry it is.",
        "awaiting": null
      }
    }
  },
  "stage": "matched",
  "generate_ready": false,
  "confirmed_destination": {
    "type": "destination",
    "id": "pondicherry_puducherry"
  }
}
        ↓
UI merges state_delta → localStorage
trip_state.stage = "matched"
trip_state.trip_context.selected_option set
        ↓
Render confirmation message
Trip Matcher complete
```

---

## Page Refresh Behaviour

```
User refreshes page
        ↓
UI reads trip_state from localStorage
        ↓
if stage = "new" or "matching":
    UI renders blank or partial chat
    POST /api/scout/message with trip_state
    Scout reads conversation_context.last_scout_message
    and awaiting to resume naturally
    message: "(resume)" or similar signal

if stage = "ready":
    UI renders Generate button
    Scout re-engages from last_scout_message

if stage = "matched":
    UI shows confirmed destination
    Can route to Planner or re-enter Matcher
```

No conversation history needed anywhere. Scout reconstructs context entirely from TripState.

---

## Error Handling

### Scout API failure

```
POST /api/scout/message fails
        ↓
UI does not update localStorage
UI shows retry prompt
User resends → retry from same trip_state
```

### Meridian infrastructure failure

```
POST /api/meridian/match fails (5xx)
        ↓
POST /api/scout/present with
  meridian_output: { "status": "API_ERROR" }
        ↓
Scout: "Hit a snag — want me to try again?"
User confirms → UI retries /api/meridian/match
```

### Meridian FAILURE status (expected, not an error)

```
Meridian returns HARD_FAIL / SOFT_FAIL / BUDGET_FAIL / CONFLICT_FAIL
        ↓
POST /api/scout/present with full failure response
Scout translates → conversational follow-up
User adjusts → refinement loop
```

---

## TripState in localStorage

Single source of truth. No conversation array. No component state for chat history.

```json
{
  "trip_id": "trip_7f3a9c",
  "status": "free",
  "stage": "ready",

  "trip_context": {
    "required_inputs": {
      "origin_city": "Bengaluru",
      "budget": 30000,
      "budget_unit": "total",
      "duration_nights": 3,
      "num_travelers": 3,
      "travel_month": "September"
    },
    "preferences": {
      "trip_goal": { "value": "Bachelorette", "confidence": "explicit" },
      "crowd_tolerance": { "value": "low", "confidence": "high" },
      "group_type": "friends",
      "travel_style": ["relaxed", "social"],
      "nuanced_preferences": [
        "Wants a chill celebration vibe",
        "Prefers boutique or aesthetic stays"
      ]
    },
    "selected_option": null
  },

  "matcher_state": {
    "generate_ready": true,
    "conversation_context": {
      "last_scout_message": "Perfect — I have everything I need. Ready when you are.",
      "awaiting": null
    },
    "last_recommendations": null
  },

  "planner_state": null
}
```

See TRIP_STATE.md for full schema, field definitions, and stage transition reference.

---

## What the Backend Does Not Own

The backend is stateless for MVP:

- Does not store TripState
- Does not store conversation history
- Does not track sessions
- Does not know if a user has called any endpoint before

Every request is self-contained. The UI sends everything the backend needs.
