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

**Nuanced Preferences**

Free-form array. Captures what the traveler expressed in their own words that doesn't map cleanly to a structured field. Scout extracts these from conversation and passes them as-is.

Examples:
- "Wants to walk a lot — prefers compact, navigable destinations"
- "Local food should be easy to find without research or apps"
- "Wants to avoid tourist-heavy pockets specifically, not just general crowd volume"
- "Prefers figuring things out spontaneously rather than having a planned itinerary"
- "Wants a place that feels local, not built for tourists"

Meridian maps these to KB experience attributes in Step 0 and uses them as a scoring modifier in Step 7.

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

**Nuanced Preference Parsing**

Map each item in `nuanced_preferences` to the closest KB experience attribute:

| Nuanced Preference Signal | KB Attribute |
|---|---|
| Walkability, exploring on foot, compact town | `experience_attributes.walkability` |
| Local food easy to find, no TripAdvisor hunting | `experience_attributes.food_culture.accessibility` |
| Feels local, not touristy | `experience_attributes.local_authenticity` |
| Avoid tourist pockets specifically | `experience_attributes.tourist_density` |
| Spontaneous, no itinerary, figure it out | `experience_attributes.spontaneity_friendly` |
| Visual character, photogenic, distinctive | `experience_attributes.visual_character` |
| Slow pace, unhurried | `experience_attributes.pace` |
| Lots to discover, explore | `experience_attributes.discovery_potential` |

Register the mapped attributes. Use them as a scoring modifier in Step 7. If a nuanced preference cannot be mapped to a KB attribute, carry it forward as a qualitative signal — do not discard it.

---

### Step 1: Candidate Generation

Query the Destination KB using semantic search.

**Duration scoping — determines what to query:**

```
1–3 nights  →  Single destination profiles only
4–5 nights  →  Single destination profiles + simple 2-stop combinations
6–8 nights  →  Circuit templates (2–3 stops) + single destinations
9–12 nights →  Circuit templates (3–4 stops) + extended circuits
```

**For single destination queries (1–3 nights):**

Search parameters: trip goal, travel style, origin city, trip duration.

**For multi-stop queries (4+ nights):**

Query circuit templates first. Match on:
- Trip goal and travel style
- `duration.minimum_nights` ≤ `duration_nights` ≤ `duration.maximum_nights`
- Origin city (to assess entry point feasibility)

Also query circuit `common_variations` for duration-appropriate sub-circuits (e.g. 4-night Rajasthan Essential instead of 7-night full circuit).

Also query individual destination profiles for simple 2-stop combinations where no circuit template exists.

Generate 8–12 candidates at this stage. More will be eliminated in subsequent steps.

If a destination or circuit is not found in the KB, trigger the generate-and-cache pipeline before proceeding. Do not skip. Do not assume.

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

### Step 5a: Travel-to-Experience Ratio
*Applies to multi-stop trips only. Skip for single destinations.*

The single biggest failure mode in multi-stop trips is spending more time in transit than at destinations. Validate that internal travel across the circuit is proportionate to the trip duration.

```
Total internal travel hours across all legs
────────────────────────────────────────── > 0.3  →  flag as tradeoff in output
   duration_nights × 16 waking hours

> 0.4  →  eliminate the circuit or suggest a trimmed variation
```

If a circuit exceeds the ratio:
- Check `common_variations` in the circuit template for a shorter version that fits
- If a valid variation exists, substitute it and note the change
- If no variation fits, eliminate the circuit

Examples:
```
5-night Rajasthan (Jaipur–Jodhpur–Udaipur)
Internal travel: ~10 hours across 2 legs
5 nights × 16 = 80 waking hours
10 / 80 = 12.5%  →  Acceptable

4-night with 3 stops + 4-hour legs each
Internal travel: ~12 hours across 3 legs
4 nights × 16 = 64 waking hours
12 / 64 = 18.75%  →  Flag as tradeoff — not eliminated but noted in cons
```

---

### Step 6: Geography Validation

Validate surviving destinations against real-world data via Maps API.

**For single destinations:**
- Verify actual travel time from origin city
- Verify available transport routes and practical connectivity
- Remove if travel time substantially exceeds theoretical estimate or if routes are unreliable in travel month

**For circuits:**
Validate the entire sequence, not just origin → first stop.

Verify:
- Origin → entry stop: actual travel time and transport options
- Each internal leg: actual travel duration, available modes, road/rail reliability in travel month
- Last stop → origin (or alternate return point): practical return journey
- Route logic: do stops flow geographically or does the sequence backtrack significantly?

Remove circuits where:
- A leg is substantially longer than the circuit template states
- Routes are unreliable or closed in the travel month (mountain passes in monsoon, flooded roads, etc.)
- The stop sequence creates unnecessary backtracking that adds travel time

---

### Step 7: Scoring and Ranking

Score only the destinations that survived all elimination steps. Do not re-score factors already used as hard filters.

**Base score — weighted factors:**

| Factor | Weight | What It Measures |
|---|---|---|
| Seasonality Fit | 30% | How well the travel month aligns with this destination's best season |
| Travel Style Fit | 25% | How closely the destination matches the stated travel style |
| Group Fit | 20% | How appropriate the destination is for the group type |
| Destination Strengths | 15% | How strongly the destination delivers on its primary appeal |
| Uniqueness | 10% | How distinct the experience is relative to other surviving candidates |

**Additional factors for circuit trips only:**

| Factor | Weight | What It Measures |
|---|---|---|
| Route Logic | 15% | How well stops flow geographically — penalise backtracking, reward logical sequencing |
| Experience Variety | 10% | How meaningfully different each stop is — three hill stations back to back scores low; forts + lakes + desert scores high |

For circuit trips, these two factors are added to the base score and all weights are re-normalised proportionally.

**Nuanced preference modifier:**

After computing the base score, apply a modifier based on how well each destination's KB experience attributes match the mapped nuanced preferences from Step 0.

```
Strong match on nuanced preferences   →  +10% on base score
Partial match                         →  No change
Significant mismatch                  →  -10% on base score
```

Example: Traveler wants walkability and accessible local food. A compact town with a strong street food scene scores +10%. A resort-heavy destination requiring a car to reach food scores -10%.

This modifier can change the ranking. A destination that scores well on base factors but conflicts with what the traveler actually cares about should rank lower than one that genuinely fits — even if its structured attributes look similar.

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

### Standard Output — Single Destination
*Returned when trip is 1–3 nights or when single destination scores higher than all circuits.*

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
Per person: ₹X,XXX – ₹X,XXX
```

**Reachability**
```
Distance:    ~XXX km from [Origin City]
Travel time: X–Y hours
Transport:   Self-drive / Train / Flight
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
Cons:
- ...
```

---

### Standard Output — Circuit Trip
*Returned when trip is 4+ nights and a circuit scores highest.*

For each circuit option (top 3, ranked):

---

#### [Circuit Name]

**Why It Matches**
```
✓ Covers [goal-relevant character]
✓ Good for [group type]
✓ [Travel month] works well for this route
✓ Stops offer genuine variety
✓ Internal travel is proportionate to time on ground
```

**Stops**

| Stop | Nights | What It Offers |
|---|---|---|
| Jaipur | 2 | Palaces, bazaars, Pink City energy |
| Jodhpur | 2 | Blue City, Mehrangarh Fort, desert edge |
| Udaipur | 2 | Lakes, refined pace, slower atmosphere |

**Internal Travel**

| Leg | Duration | Mode |
|---|---|---|
| Jaipur → Jodhpur | 5–6 hours | Train or cab |
| Jodhpur → Udaipur | 4–5 hours | Train or cab |

**Budget Expectations**
```
Stay:            ₹XX,XXX – ₹XX,XXX  (across all stops)
Food:            ₹XX,XXX – ₹XX,XXX
Internal travel: ₹XX,XXX – ₹XX,XXX
Entry to origin: ₹XX,XXX – ₹XX,XXX
Total:           ₹XX,XXX – ₹XX,XXX
Per person:      ₹X,XXX – ₹X,XXX
```

**Entry to Circuit**
```
From [Origin City] to [First Stop]:
Duration / Transport / Cost estimate
```

**Return from Circuit**
```
From [Last Stop] to [Origin City]:
Duration / Transport / Cost estimate
```

**Weather in [Travel Month]**
```
Circuit-level seasonal character — note any variation across stops
```

**Crowd Expectations**
```
Per stop if meaningfully different, otherwise circuit-level
```

**Tradeoffs**
```
Pros:
- ...
Cons:
- ...
```

---

**Final Recommendation**

`Best Match` — [Destination or Circuit]: Why it ranks first for this specific trip.

`Alternative 1` — [Destination or Circuit]: What different experience or pace it offers.

`Alternative 2` — [Destination or Circuit]: What different need it serves.

---

**Refinement Hooks**
*Returned to Scout. Not shown to the traveler.*

```
weakest_scoring_factor: [factor name]
constraint_with_highest_elimination: [constraint]
budget_headroom: tight / comfortable / flexible
seasonality_note: [any flag if travel month is suboptimal]
nuanced_preference_gaps: [any nuanced preferences that no option satisfied well]
travel_to_experience_flag: [flag if any circuit was close to the 0.3 ratio threshold]
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
