# Trip Matcher — Technical API Flow

## Overview

Two endpoints. One for Scout, one for Meridian.

The UI calls Scout for every user interaction. It calls Meridian only when the user taps Generate. Scout reads TripState on every call — no conversation history, no session state. TripState is the single source of truth.

---

## Backend Architecture

```
UI (Browser)
    ↕
Backend (single service)
    ├── POST /api/scout      →  Anthropic API (Scout system prompt)
    └── POST /api/meridian   →  Anthropic API (Meridian system prompt)
                                + Supabase (KB query)
```

---

## POST /api/scout

Handles all Scout interactions. One endpoint, one request shape, one response shape.

`message: null` signals Meridian has just run — Scout reads `last_recommendations` from TripState and presents. `message: string` is a regular user turn. Scout infers what to do from TripState alone.

### Request

```json
{
  "trip_state": { },
  "message": "string | null"
}
```

`trip_state` — full current TripState from localStorage. Always sent in its entirety.

`message` — the user's latest message, or `null` if this is a post-Meridian presentation call.

#### Example — Regular user turn

```json
{
  "trip_state": {
    "trip_id": "trip_7f3a9c",
    "status": "free",
    "stage": "matching",
    "trip_context": {
      "required_inputs": {
        "origin_city": "Bengaluru",
        "budget": 30000,
        "budget_unit": "total",
        "duration_nights": null,
        "num_travelers": null,
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
  "message": "30k total, 3 nights, we are 4 people"
}
```

#### Example — Post-Meridian presentation

UI has already written Meridian output to `matcher_state.last_recommendations`. Then calls Scout with `message: null`:

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
        "num_travelers": 4,
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
      "generate_ready": false,
      "conversation_context": {
        "last_scout_message": "Perfect — ready when you are.",
        "awaiting": null
      },
      "last_recommendations": {
        "options": [
          "pondicherry_puducherry",
          "hampi_karnataka",
          "goa_north"
        ],
        "best_match": "pondicherry_puducherry",
        "generated_at": "2025-09-15T10:45:00Z",
        "meridian_output": { "...full Meridian JSON..." }
      }
    },
    "planner_state": null
  },
  "message": null
}
```

---

### Response

Same shape always. Everything lives in `state_delta`. UI deep-merges into TripState and writes to localStorage.

```json
{
  "message": "string",
  "state_delta": {
    "stage": "string | omit if unchanged",
    "trip_context": {
      "required_inputs": { "...only changed fields..." },
      "preferences": { "...only new/updated fields..." },
      "selected_option": { "...or omit if unchanged..." }
    },
    "matcher_state": {
      "generate_ready": "boolean | omit if unchanged",
      "conversation_context": {
        "last_scout_message": "string",
        "awaiting": "string | null"
      }
    }
  }
}
```

Only changed fields appear in `state_delta`. UI merges — does not replace. If a section has no changes, omit it entirely.

#### Example responses

**Regular turn — still collecting:**

```json
{
  "message": "Got it — ₹30,000 total, 3 nights, 4 people from Bengaluru. Is there anything else you'd like me to factor in? Travel style, crowd preferences, anything specific?",
  "state_delta": {
    "stage": "matching",
    "trip_context": {
      "required_inputs": {
        "budget": 30000,
        "budget_unit": "total",
        "duration_nights": 3,
        "num_travelers": 4
      },
      "preferences": {}
    },
    "matcher_state": {
      "generate_ready": false,
      "conversation_context": {
        "last_scout_message": "Got it — ₹30,000 total, 3 nights, 4 people from Bengaluru. Is there anything else you'd like me to factor in?",
        "awaiting": "additional_preferences"
      }
    }
  }
}
```

**Ready signal:**

```json
{
  "message": "Perfect — I have everything I need. Ready to find recommendations when you are.",
  "state_delta": {
    "stage": "ready",
    "trip_context": {
      "required_inputs": {},
      "preferences": {
        "nuanced_preferences": ["Wants a chill celebration vibe, not a big party scene"]
      }
    },
    "matcher_state": {
      "generate_ready": true,
      "conversation_context": {
        "last_scout_message": "Perfect — I have everything I need. Ready to find recommendations when you are.",
        "awaiting": null
      }
    }
  }
}
```

**Presentation (message: null):**

```json
{
  "message": "Found three options. Pondicherry comes up top — fits your ₹30,000 budget cleanly at about ₹6,000 per person, good bachelorette vibe with boutique cafés and a relaxed beach scene, quieter than usual in September. Hampi is an interesting alternative if you want something more photogenic and offbeat with budget headroom. Goa is possible but tight — September and the budget make it a stretch. Want me to walk through any of these in detail?",
  "state_delta": {
    "stage": "ready",
    "matcher_state": {
      "generate_ready": false,
      "conversation_context": {
        "last_scout_message": "Found three options. Pondicherry comes up top...",
        "awaiting": "user_response_to_recommendations"
      }
    }
  }
}
```

**Destination confirmed:**

```json
{
  "message": "Pondicherry it is. Great choice for a September bachelorette trip.",
  "state_delta": {
    "stage": "matched",
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
        "last_scout_message": "Pondicherry it is. Great choice for a September bachelorette trip.",
        "awaiting": null
      }
    }
  }
}
```

**Failure presentation (message: null, Meridian returned FAIL):**

```json
{
  "message": "Nothing came through that matched everything — the main conflict is that September is tricky for crowd-free options near Bengaluru. Most quiet places in September are quiet because of heavy rain. Would you be open to October instead? The crowd situation stays similar but the weather improves a lot.",
  "state_delta": {
    "stage": "matching",
    "matcher_state": {
      "generate_ready": false,
      "conversation_context": {
        "last_scout_message": "Nothing came through that matched everything...",
        "awaiting": "constraint_relaxation"
      }
    }
  }
}
```

---

## POST /api/meridian

Called on button tap only. Returns recommendations. Never called by Scout.

### Request

```json
{
  "trip_context": {
    "required_inputs": {
      "origin_city": "Bengaluru",
      "budget": 30000,
      "budget_unit": "total",
      "duration_nights": 3,
      "num_travelers": 4,
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

UI sends `trip_state.trip_context` directly. No wrapping needed.

### Response (SUCCESS)

```json
{
  "status": "SUCCESS",
  "trip_type": "single",
  "budget_basis": {
    "total": 30000,
    "per_person": 7500,
    "num_travelers": 4
  },
  "options": [
    {
      "rank": 1,
      "type": "single",
      "name": "Pondicherry (Puducherry)",
      "destination_id": "pondicherry_puducherry",
      "match_sections": [
        {
          "type": "budget",
          "heading": "Realistic budget breakdown (3 nights, 4 people)",
          "stay": {
            "notes": "Boutique guesthouse near promenade",
            "estimate_per_person": 2625,
            "estimate_group": 10500,
            "assumption": "avg ~3,500 INR/night for whole unit"
          },
          "food": {
            "notes": "Cafés, beachside dinners, mix of mid-range and cheap eats",
            "estimate_per_person": 1200,
            "estimate_group": 4800,
            "assumption": "~400 INR/day/person"
          },
          "travel": {
            "notes": "AC Volvo / private shared taxi from Bengaluru",
            "estimate_per_person": 1200,
            "estimate_group": 4800,
            "assumption": "1,000–1,500 INR per person roundtrip by bus"
          },
          "activities": {
            "notes": "Auroville visit, beach time, photography spots",
            "estimate_per_person": 500,
            "estimate_group": 2000,
            "assumption": "light paid activities"
          },
          "local_transport": {
            "notes": "Scooter rental / tuk-tuks",
            "estimate_per_person": 500,
            "estimate_group": 2000,
            "assumption": "scooter ₹300–₹500/day shared"
          },
          "per_person_total": 6025,
          "group_total": 24100,
          "budget_given": 30000,
          "verdict": "comfortable"
        },
        {
          "type": "trip_goal",
          "heading": "Bachelorette fit",
          "points": [
            "Boutique cafés, pastel streets, and beachfront promenades for photos and small celebrations",
            "Private guesthouse options for group privacy and in-house gatherings",
            "A few lively bars ideal for a 4-person group — celebratory without overwhelming"
          ]
        },
        {
          "type": "crowd_preference",
          "heading": "Quieter than usual in September",
          "points": [
            "Shoulder season — town is quieter than peak",
            "Some weekend pockets busier but overall not crowded",
            "More intimate than peak-season Goa"
          ]
        },
        {
          "type": "weather",
          "heading": "Some rain expected — plan flexibly",
          "contextual": true,
          "points": [
            "Tail of monsoon — warm, humid, occasional showers",
            "Rain usually short bursts — beach time still possible",
            "Auroville stays lush and photogenic after rain"
          ]
        },
        {
          "type": "reachability",
          "heading": "Practical from Bengaluru",
          "points": [
            "~6–7 hours by road — AC bus or shared taxi",
            "Overnight bus saves daytime for the trip",
            "No flight needed for a 3-night trip"
          ]
        }
      ],
      "tradeoffs": [
        {
          "point": "Nightlife is low-key — better for intimate celebrations than large party nights",
          "affects": "trip_goal"
        },
        {
          "point": "September rain can interrupt beach plans on short notice",
          "affects": "weather"
        }
      ]
    }
  ],
  "final_recommendation": {
    "best_match": "Pondicherry (Puducherry)",
    "best_match_reason": "Best balance of bachelorette vibe, manageable travel, budget comfort, and September quiet",
    "alternative_1": "Hampi (Karnataka)",
    "alternative_1_reason": "Most budget-friendly, highly photogenic, great for a creative intimate bachelorette — leaves headroom for extras",
    "alternative_2": "Goa (North Goa — budget approach)",
    "alternative_2_reason": "Classic choice — possible within budget only on bus + budget stays. September lowers crowds but also limits nightlife"
  },
  "refinement_hooks": {
    "weakest_scoring_factor": "Nightlife intensity — options vary in party energy; Pondicherry and Hampi are mellow vs peak-season Goa",
    "constraint_with_highest_elimination": "Budget (₹30,000 total) — rules out flying and mid-range stays for Goa",
    "budget_headroom": "comfortable",
    "seasonality_note": "September is monsoon tail — expect rain and humidity. Hampi and Pondicherry stay photogenic; Goa's beach nights are unreliable",
    "nuanced_preference_gaps": null
  }
}
```

### Response (FAILURE)

```json
{
  "status": "HARD_FAIL",
  "message": "No destinations matched all the stated constraints",
  "eliminating_constraints": ["crowd_tolerance: low", "travel_month: September"],
  "relaxation_suggestions": [
    "Consider October — crowds similar but rain significantly lower",
    "Moderate crowd tolerance to balanced"
  ],
  "surviving_destinations": []
}
```

---

## Full UI Flow

### Input Collection Loop

```
User sends message
        ↓
POST /api/scout
  body: { trip_state, message: "..." }
        ↓
Deep-merge state_delta into trip_state → localStorage
Render response.message in chat
        ↓
if state_delta.matcher_state.generate_ready = false:
    Wait for next message → loop

if state_delta.matcher_state.generate_ready = true:
    Show "Generate Recommendations" button
    Wait for user tap
```

---

### User Taps Generate

```
User taps "Generate Recommendations"
        ↓
Button enters loading state
        ↓
POST /api/meridian
  body: { trip_context: trip_state.trip_context }
        ↓
Write full Meridian response to
  trip_state.matcher_state.last_recommendations
  → localStorage
        ↓
POST /api/scout
  body: { trip_state (with last_recommendations populated), message: null }
        ↓
Deep-merge state_delta → localStorage
Render Scout's presentation in chat
Hide Generate button
```

Meridian output — SUCCESS or FAILURE — always goes to `last_recommendations` before calling Scout. Scout handles both. UI does not need to distinguish between success and failure before calling Scout.

---

### Refinement

```
User responds with adjustment
        ↓
POST /api/scout
  body: { trip_state, message: "What if we go in October instead?" }
        ↓
Scout extracts adjustment → state_delta
Deep-merge → localStorage
        ↓
if generate_ready = true → show Generate button
User taps → POST /api/meridian → POST /api/scout (message: null)
        ↓
Updated recommendations shown
```

---

### Confirmation

```
User confirms destination
        ↓
POST /api/scout
  body: { trip_state, message: "Let's go with Pondicherry" }
        ↓
Scout returns:
  state_delta.trip_context.selected_option = { type, id }
  state_delta.stage = "matched"
        ↓
Deep-merge → localStorage
stage = "matched"
Trip Matcher complete
```

---

### Page Refresh

```
User refreshes
        ↓
UI reads trip_state from localStorage
        ↓
stage = "new" or "matching":
    POST /api/scout
      body: { trip_state, message: null }
    Scout reads conversation_context and resumes:
    "Welcome back — you were planning a bachelorette
     trip from Bengaluru. Still thinking September?"

stage = "ready":
    Show Generate button
    POST /api/scout
      body: { trip_state, message: null }
    Scout re-engages from last_scout_message

stage = "matched":
    Show confirmed destination
    No Scout call needed unless user re-enters Matcher
```

---

## Error Handling

### Scout failure

```
POST /api/scout fails (5xx / network)
        ↓
UI does not update localStorage
UI shows: "Something went wrong — try again"
User resends → retry with same trip_state
```

### Meridian infrastructure failure

```
POST /api/meridian fails (5xx)
        ↓
Write { status: "API_ERROR" } to last_recommendations
POST /api/scout
  body: { trip_state, message: null }
Scout: "Hit a snag — want to try again?"
User confirms → UI retries /api/meridian
```

### Meridian FAILURE status (expected — not an error)

```
Meridian returns HARD_FAIL / SOFT_FAIL / BUDGET_FAIL / CONFLICT_FAIL
        ↓
Write failure response to last_recommendations
POST /api/scout
  body: { trip_state, message: null }
Scout translates → natural follow-up question
User adjusts → refinement loop
```

UI never needs to distinguish Meridian SUCCESS from FAILURE. Always write to `last_recommendations`, always call Scout. Scout handles it.

---

## TripState Reference

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
      "num_travelers": 4,
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

See TRIP_STATE.md for full schema, field definitions, and stage transitions.

---

## What the Backend Does Not Own

Stateless for MVP:
- No TripState storage
- No conversation history
- No session tracking
- Every request is self-contained
