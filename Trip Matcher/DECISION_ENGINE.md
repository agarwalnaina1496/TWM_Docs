# Meridian — Decision Engine

## Overview

Meridian is TWM's destination recommendation engine. It receives a structured input payload from Scout (the Conversation Agent), runs a multi-step elimination and scoring pipeline, and returns a ranked shortlist of destinations with full reasoning and refinement signals.

Meridian is stateless. It holds no conversation history. It processes one complete input payload per run and returns one complete output payload. All decision-making logic lives here. Scout does not recommend — Meridian does.

---

## Inputs

### Required Inputs
These must be present in every payload. If any are missing, Meridian returns a `MISSING_INPUTS` signal to Scout.

| Input | Description |
|---|---|
| Budget | Total trip budget in ₹ |
| Travel Duration | Number of nights |
| Travel Month | Month of travel (for seasonality scoring) |
| Number of Travelers | Integer |
| Origin City | Departure city |

### Recommended Inputs
Collected progressively by Scout. The more that are present, the more precise the output.

**Trip Goal**
- Relaxation
- Celebration (Birthday / Anniversary / etc.)
- Adventure
- Staycation
- Workation

**Travel Style**
- Scenic
- Relaxed
- Nature
- Cultural
- Adventure
- Luxury
- Social
- Road Trip

**Crowd Preference**
- Avoid crowds
- Balanced
- Love lively destinations

**Weather Preference**
- Cool weather
- Warm weather
- Snow
- Monsoon
- Avoid rain
- Pleasant temperatures

**Group Type**
- Solo
- Couple
- Friends
- Family

**Budget Flexibility**
- Strict
- Moderate
- Not an issue

**Explicit Exclusions**
- No beaches
- No mountains
- No nightlife
- No trekking
- No long journeys
- No flights
- No international travel
- Any other explicit mention

---

## Decision Engine

### Step 0: Preference and Override Parsing

Before running any logic, parse and register all explicit user preferences. These override system defaults at every subsequent step.

Priority hierarchy:
```
Explicit User Preference > System Defaults > Fallback Heuristics
```

**Budget Handling**

"Budget is not an issue" → interpret as `Moderate` flexibility, not Luxury.

Luxury travel only if user explicitly states:
- Luxury only
- Premium stays preferred
- Happy to spend whatever it takes
- Private transfers are fine

**Transport Preferences**

If user specifies transport, it overrides all transport logic in Steps 4 and 5:
- Flight only
- Train only
- Road trip / self-drive

**Travel Time Preferences**

If user specifies travel time, it overrides the defaults in Step 5:
- "Prefer within 6 hours"
- "Happy to travel 20+ hours"

---

### Step 1: Candidate Generation

Query the Destination KB using semantic search.

Search parameters:
- Trip goal
- Travel style
- Origin city (for travel context)
- Trip duration

Candidates scoped by trip duration:
```
2–3 nights  →  Single destination
4–5 nights  →  Single or dual destination
6–8 nights  →  Multi-stop or circuit trips
```

If a queried destination is not found in the KB, trigger the generate-and-cache pipeline before proceeding. Do not skip. Do not assume.

---

### Step 2: Hard Constraint Elimination

Remove any destination that violates an explicit user exclusion.

Examples:
- No beaches → remove coastal destinations
- No trekking → remove destinations where trekking is the primary activity
- No flights → remove flight-only reachable destinations
- Avoid rain → check travel month against destination's monsoon and rain profile
- Cool weather preferred → remove destinations with high temperatures in the travel month

---

### Step 3: Goal Fit Elimination

Remove destinations fundamentally incompatible with the stated trip goal.

```
Relaxation      →  Remove high-stimulation, activity-centric destinations
Celebration     →  Remove destinations without social or nightlife options
Adventure       →  Remove purely relaxation-oriented destinations
Wildlife        →  Remove destinations with weak wildlife credentials
Beach Holiday   →  Remove non-coastal destinations
Workation       →  Remove destinations with poor connectivity
```

---

### Step 4: Transport Feasibility
*Skip if user has specified a transport preference.*

Typical domestic round-trip flights: ₹10,000–₹20,000 per person.

Evaluate whether flight cost would materially reduce the overall trip experience given total budget and number of travelers.

- If flights are reasonable → flight routes remain in play
- If not → prefer train / bus / cab / self-drive reachable destinations

Do not eliminate destinations purely because they are flight-accessible.

---

### Step 5: Travel Time Feasibility
*Skip if user has specified a travel time preference.*

Default limits:
- Maximum one-way travel time: 15 hours
- Maximum transfers: 2

Remove destinations that exceed these limits from the origin city.

---

### Step 6: Geography Validation

Validate surviving destinations against real-world data via Maps API.

Verify:
- Actual travel time from origin city
- Available transport routes
- Practical connectivity

Remove destinations that appear feasible in theory but are not practical in reality.

---

### Step 7: Scoring and Ranking

Score only the destinations that survived all elimination steps. Do not re-score factors already used as hard filters.

| Factor | Weight | What It Measures |
|---|---|---|
| Seasonality Fit | 30% | How well the travel month aligns with this destination's best season |
| Travel Style Fit | 25% | How closely the destination matches the stated travel style |
| Group Fit | 20% | How appropriate the destination is for the group type |
| Destination Strengths | 15% | How strongly the destination delivers on its primary appeal |
| Uniqueness | 10% | How distinct the experience is relative to other surviving candidates |

---

### Step 8: Failure State Detection

Before returning output, check for failure conditions.

| Condition | Signal |
|---|---|
| Zero destinations survive elimination | `HARD_FAIL` |
| Fewer than 3 destinations survive | `SOFT_FAIL` |
| Budget insufficient for all candidates | `BUDGET_FAIL` |
| User constraints are mutually exclusive | `CONFLICT_FAIL` |
| Required inputs missing from payload | `MISSING_INPUTS` |

See Failure Output section for return format.

---

## Output

### Standard Output
*Returned when 3 or more destinations survive.*

For each destination (top 3, ranked):

---

#### [Destination Name]

**Why It Matches**
```
✓ Scenic
✓ Relaxed
✓ Good for couples
✓ Great in [travel month]
✓ Low crowds
```

**Budget Expectations**
```
Stay:    ₹X,XXX – ₹X,XXX
Food:    ₹X,XXX – ₹X,XXX
Travel:  ₹X,XXX – ₹X,XXX
Total:   ₹XX,XXX – ₹XX,XXX
```

**Reachability**
```
Distance:   ~XXX km from [Origin City]
Travel time: X–Y hours
Transport:  Self-drive / Train / Flight
```

**Weather in [Travel Month]**
```
Character and what the traveler should expect
```

**Crowd Expectations**
```
Low / Moderate / High
Context specific to this destination and month
```

**Tradeoffs**
```
Pros:
- ...
- ...

Cons:
- ...
- ...
```

---

**Final Recommendation**

`Best Match` — [Destination]: Why it ranks first relative to this traveler's specific goal.

`Alternative 1` — [Destination]: What different need or style it serves.

`Alternative 2` — [Destination]: What different experience it offers.

---

**Refinement Hooks**
*Returned to Scout. Not shown to the traveler.*

```
weakest_scoring_factor: [factor name]
constraint_with_highest_elimination: [constraint]
budget_headroom: tight / comfortable / flexible
seasonality_note: [any flag if travel month is suboptimal]
```

---

### Failure Output

```json
{
  "status": "HARD_FAIL | SOFT_FAIL | BUDGET_FAIL | CONFLICT_FAIL | MISSING_INPUTS",
  "message": "Human-readable explanation for Scout to translate",
  "eliminating_constraints": ["constraint_1", "constraint_2"],
  "relaxation_suggestions": ["Suggestion 1", "Suggestion 2"],
  "minimum_viable_budget": 45000,
  "conflicting_constraints": ["avoid_rain", "travel_month: July"],
  "missing_fields": ["travel_month"],
  "surviving_destinations": ["Destination A"]
}
```

All fields are optional except `status` and `message`. Return only the fields relevant to the failure type.
