# Destination Knowledge Base — Schema and Strategy

## Overview

The Destination KB is the data layer that Meridian queries during candidate generation and scoring. It stores structured destination profiles optimised for TWM's elimination and scoring logic.

The KB is not a static document. It is a living database that starts seeded and grows dynamically as new destinations are queried.

---

## Why a Dedicated KB

Meridian's Decision Engine needs structured, consistent inputs to run elimination and scoring reliably. Web search returns unstructured, variable results — parsing these into decision-engine-compatible data at query time introduces inconsistency and latency at the core of the product.

The KB solves this by providing:
- Structured profiles Meridian can query directly
- Consistent field definitions across all destinations
- Seasonality, crowd, cost, and fit data in a form the scoring logic can use
- A layer that improves over time as usage grows

Live data that changes frequently — current pricing, exact travel times, live transport options — is always fetched at query time from web or Maps APIs. The KB covers what a destination *is*. Live sources cover what it currently *costs and takes to get to*.

---

## Destination Profile Schema

```json
{
  "id": "coorg_karnataka",
  "name": "Coorg",
  "aliases": ["Kodagu", "Scotland of India"],
  "region": "Karnataka, South India",
  "destination_type": ["hill station", "nature", "plantation"],

  "seasonality": {
    "best_months": ["October", "November", "December", "January", "February", "March"],
    "acceptable_months": ["April", "May", "September"],
    "avoid_months": ["June", "July", "August"],
    "peak_tourist_months": ["December", "January"],
    "off_peak_months": ["June", "July", "August", "September"],
    "monsoon_character": "Heavy rain from June to August. Lush and scenic but some activities restricted. Roads can be difficult."
  },

  "trip_goal_fit": {
    "relaxation": "high",
    "adventure": "medium",
    "celebration": "low",
    "workation": "medium",
    "staycation": "high"
  },

  "travel_style_fit": {
    "scenic": "high",
    "nature": "high",
    "relaxed": "high",
    "cultural": "medium",
    "adventure": "medium",
    "luxury": "medium",
    "social": "low",
    "road_trip": "high"
  },

  "group_fit": {
    "solo": "medium",
    "couple": "high",
    "friends": "high",
    "family": "high"
  },

  "crowd_profile": {
    "typical_level": "moderate",
    "peak_level": "high",
    "off_peak_level": "low",
    "notes": "Busy during December-January long weekends. Quieter on weekdays and in shoulder months."
  },

  "cost": {
    "tier": "mid",
    "typical_per_person_per_day": {
      "budget": 1500,
      "mid": 3000,
      "premium": 6000
    },
    "notes": "Homestays available at budget end. Premium resorts and estate stays at upper end."
  },

  "reachability": {
    "bengaluru": {
      "distance_km": 250,
      "typical_duration_hours": "5-6",
      "transport_options": ["self-drive", "bus", "cab"],
      "notes": "Well-connected by road. No direct train. KSRTC buses available."
    },
    "chennai": {
      "distance_km": 520,
      "typical_duration_hours": "9-10",
      "transport_options": ["self-drive", "cab"],
      "notes": "Long drive. Not ideal for a 2-night trip from Chennai."
    }
  },

  "destination_profile": {
    "known_for": ["coffee and spice plantations", "mist and greenery", "waterfalls", "homestays", "Abbey Falls", "Raja's Seat"],
    "primary_activities": ["plantation walks", "waterfall visits", "resort and homestay stays", "estate tours", "local market visits"],
    "not_suitable_for": ["nightlife", "beach", "urban experiences", "large-group party trips"],
    "trekking_required": false,
    "beach": false,
    "mountains": true,
    "wildlife": "low",
    "nightlife": "low",
    "connectivity": "good"
  },

  "destination_strengths": [
    "Scenic beauty year-round outside monsoon",
    "Peaceful, unhurried atmosphere",
    "Strong homestay culture — feels local",
    "Very accessible from Bengaluru for a 2-3 night trip"
  ],

  "experience_attributes": {
    "walkability": "low",
    "navigability": "low",
    "food_culture": {
      "quality": "medium",
      "accessibility": "medium",
      "street_food": false,
      "notes": "Good food within stays and homestays. Standalone restaurants limited outside main town. Not a destination you can wander into food."
    },
    "local_authenticity": "high",
    "tourist_density": "moderate",
    "spontaneity_friendly": "medium",
    "visual_character": "Misty hills, coffee and spice plantations, waterfalls, dense greenery",
    "pace": "slow",
    "discovery_potential": "medium"
  },

  "uniqueness_notes": "One of the most accessible and well-rounded hill escapes from Bengaluru. Distinct plantation character sets it apart from standard hill stations.",

  "meta": {
    "last_updated": "2025-10-01",
    "data_source": "curated",
    "confidence": "high"
  }
}
```

---

## Circuit Template Schema

Circuit templates are a separate entity in the KB. They are not assembled by Meridian on the fly — they are pre-defined, reviewed structures that Meridian queries directly for multi-stop trips.

```json
{
  "id": "rajasthan_classic",
  "name": "Rajasthan Classic Circuit",
  "type": "circuit",
  "structure": "linear",

  "stops": [
    {
      "destination_id": "jaipur_rajasthan",
      "name": "Jaipur",
      "recommended_nights": { "min": 2, "optimal": 3 },
      "role": "Gateway city — palaces, bazaars, Pink City character",
      "experience_type": ["cultural", "architectural"]
    },
    {
      "destination_id": "jodhpur_rajasthan",
      "name": "Jodhpur",
      "recommended_nights": { "min": 1, "optimal": 2 },
      "role": "Blue city — Mehrangarh Fort, desert edge feel",
      "experience_type": ["cultural", "architectural", "desert"]
    },
    {
      "destination_id": "udaipur_rajasthan",
      "name": "Udaipur",
      "recommended_nights": { "min": 2, "optimal": 3 },
      "role": "Lake city — refined pace, romantic, ideal end point",
      "experience_type": ["cultural", "scenic", "relaxed"]
    }
  ],

  "internal_travel": [
    {
      "from": "Jaipur",
      "to": "Jodhpur",
      "distance_km": 340,
      "typical_duration_hours": "5-6",
      "modes": ["train", "cab"],
      "notes": "Several trains daily. Cab comfortable for flexible timing."
    },
    {
      "from": "Jodhpur",
      "to": "Udaipur",
      "distance_km": 250,
      "typical_duration_hours": "4-5",
      "modes": ["train", "cab"],
      "notes": "Scenic drive through desert landscape."
    }
  ],

  "duration": {
    "minimum_nights": 5,
    "optimal_nights": 7,
    "maximum_nights": 10
  },

  "seasonality": {
    "best_months": ["October", "November", "December", "January", "February", "March"],
    "acceptable_months": ["September", "April"],
    "avoid_months": ["May", "June", "July", "August"],
    "notes": "Peak heat May–June makes sightseeing brutal. Monsoon manageable but dusty and humid."
  },

  "circuit_character": "History, Mughal and Rajput architecture, desert landscape, vibrant bazaars, lake towns",

  "experience_variety": "high",
  "experience_variety_notes": "Each stop offers a meaningfully different character — forts and markets in Jaipur, desert edge in Jodhpur, lakes and romance in Udaipur.",

  "trip_goal_fit": {
    "cultural": "high",
    "relaxation": "medium",
    "adventure": "low",
    "celebration": "medium",
    "workation": "low"
  },

  "travel_style_fit": {
    "cultural": "high",
    "scenic": "high",
    "luxury": "high",
    "relaxed": "medium",
    "social": "medium",
    "nature": "low",
    "adventure": "low"
  },

  "group_fit": {
    "couple": "high",
    "family": "high",
    "friends": "medium",
    "solo": "high"
  },

  "cost": {
    "tier": "mid",
    "typical_total_per_person": {
      "budget": 15000,
      "mid": 30000,
      "premium": 70000
    },
    "notes": "Heritage hotels and havelis significantly raise costs. Budget guesthouses widely available."
  },

  "known_pinch_points": [
    "Internal legs add up — each is a half-day commitment",
    "December–January peak: accommodation books fast, prices spike",
    "Adding Jaisalmer significantly increases internal travel burden"
  ],

  "common_variations": [
    {
      "name": "Rajasthan Essential",
      "stops": ["Jaipur", "Udaipur"],
      "nights": "4-5",
      "notes": "Best when duration doesn't allow all three stops"
    },
    {
      "name": "Rajasthan Extended",
      "stops": ["Jaipur", "Jodhpur", "Udaipur", "Jaisalmer"],
      "nights": "9-12",
      "notes": "Only viable for 9+ nights given internal travel"
    }
  ],

  "meta": {
    "last_updated": "2025-10-01",
    "data_source": "curated",
    "confidence": "high"
  }
}
```

---

## Field Definitions

| Field | Type | Purpose in Meridian |
|---|---|---|
| `destination_type` | array | Step 1 candidate generation |
| `best_months / avoid_months` | array | Step 2 (if avoid rain), Step 7 seasonality scoring |
| `peak_tourist_months` | array | Step 2 (if avoid crowds) |
| `trip_goal_fit` | object | Step 3 goal fit elimination and Step 7 scoring |
| `travel_style_fit` | object | Step 7 travel style scoring |
| `group_fit` | object | Step 7 group fit scoring |
| `crowd_profile` | object | Step 2 crowd filter and Step 7 crowd context |
| `cost.tier` | string | Step 1 budget-tier filtering |
| `cost.typical_per_person_per_day` | object | Output budget expectations |
| `reachability` | object | Step 5 travel time feasibility |
| `trekking_required` | boolean | Step 2 (if no trekking exclusion) |
| `beach / mountains / nightlife` | boolean | Step 2 explicit exclusion matching |
| `connectivity` | string | Step 3 (workation goal fit) |
| `destination_strengths` | array | Step 7 destination strengths scoring |
| `uniqueness_notes` | string | Step 7 uniqueness scoring |
| `experience_attributes.walkability` | string | Step 7 nuanced preference modifier |
| `experience_attributes.navigability` | string | Step 7 nuanced preference modifier |
| `experience_attributes.food_culture.accessibility` | string | Step 7 nuanced preference modifier |
| `experience_attributes.food_culture.quality` | string | Step 7 nuanced preference modifier |
| `experience_attributes.local_authenticity` | string | Step 7 nuanced preference modifier |
| `experience_attributes.tourist_density` | string | Step 7 nuanced preference modifier |
| `experience_attributes.spontaneity_friendly` | string | Step 7 nuanced preference modifier |
| `experience_attributes.visual_character` | string | Step 7 nuanced preference modifier, output |
| `experience_attributes.pace` | string | Step 7 nuanced preference modifier |
| `experience_attributes.discovery_potential` | string | Step 7 nuanced preference modifier |

**Circuit Template Fields**

| Field | Type | Purpose in Meridian |
|---|---|---|
| `stops` | array | Step 1 candidate generation, output structure |
| `internal_travel` | array | Step 5 travel ratio check, Step 6 route validation, output |
| `duration.minimum_nights / optimal_nights` | integer | Step 1 duration scoping |
| `circuit_character` | string | Step 1 semantic search, Step 7 goal fit |
| `experience_variety` | string | Step 7 variety scoring |
| `trip_goal_fit` | object | Step 3 goal fit elimination, Step 7 scoring |
| `travel_style_fit` | object | Step 7 travel style scoring |
| `group_fit` | object | Step 7 group fit scoring |
| `seasonality.best_months / avoid_months` | array | Step 2 constraint check, Step 7 seasonality scoring |
| `known_pinch_points` | array | Output cons, refinement hooks |
| `common_variations` | array | Step 1 variation matching by duration |
| `cost.typical_total_per_person` | object | Step 1 budget filtering, output budget expectations |

---

## Circuit Template Schema

Circuit templates are a separate KB entity from destination profiles. They define pre-validated multi-stop trips — stop sequences, internal travel, duration windows, and circuit-level character.

```json
{
  "id": "rajasthan_classic",
  "name": "Rajasthan Classic Circuit",
  "aliases": ["Rajasthan Triangle", "Royal Rajasthan"],
  "circuit_type": "linear",

  "stops": [
    {
      "order": 1,
      "name": "Jaipur",
      "recommended_nights": 2,
      "min_nights": 1,
      "role": "Gateway city — palaces, bazaars, Pink City energy"
    },
    {
      "order": 2,
      "name": "Jodhpur",
      "recommended_nights": 2,
      "min_nights": 1,
      "role": "Blue City — Mehrangarh Fort, desert edge"
    },
    {
      "order": 3,
      "name": "Udaipur",
      "recommended_nights": 2,
      "min_nights": 2,
      "role": "Lake City — refined pace, ideal end point"
    }
  ],

  "internal_travel": [
    {
      "from": "Jaipur",
      "to": "Jodhpur",
      "distance_km": 340,
      "typical_duration_hours": "5-6",
      "recommended_mode": ["train", "cab"],
      "notes": "Direct trains available. Cab is a comfortable alternative."
    },
    {
      "from": "Jodhpur",
      "to": "Udaipur",
      "distance_km": 250,
      "typical_duration_hours": "4-5",
      "recommended_mode": ["train", "cab"],
      "notes": "Scenic drive through Aravalli hills."
    }
  ],

  "duration": {
    "minimum_nights": 5,
    "optimal_nights": 6,
    "maximum_nights": 9
  },

  "gateway_connections": {
    "Delhi": {
      "to_first_stop": "Jaipur",
      "typical_duration": "4-5 hours",
      "recommended_mode": ["train", "flight", "cab"]
    },
    "Mumbai": {
      "to_first_stop": "Jaipur",
      "typical_duration": "2 hours",
      "recommended_mode": ["flight"]
    }
  },

  "return_connection": {
    "from_last_stop": "Udaipur",
    "typical_options": [
      { "to": "Delhi", "duration": "1.5 hours", "mode": "flight" },
      { "to": "Mumbai", "duration": "1.5 hours", "mode": "flight" }
    ]
  },

  "common_variations": [
    {
      "name": "Rajasthan Essential",
      "stops": ["Jaipur", "Udaipur"],
      "duration_nights": "3-4",
      "notes": "Drops Jodhpur for shorter trips"
    },
    {
      "name": "Rajasthan Extended",
      "stops": ["Jaipur", "Jodhpur", "Jaisalmer", "Udaipur"],
      "duration_nights": "9-12",
      "notes": "Adds Jaisalmer desert leg"
    }
  ],

  "seasonality": {
    "best_months": ["October", "November", "December", "January", "February", "March"],
    "acceptable_months": ["April"],
    "avoid_months": ["May", "June", "July", "August", "September"],
    "circuit_seasonal_notes": "Summer is extremely hot (45°C+). Monsoon brings rain to eastern Rajasthan. October–March is the ideal window."
  },

  "circuit_character": "Regal history, Mughal and Rajput architecture, desert landscapes, vibrant bazaars",

  "trip_goal_fit": {
    "relaxation": "low",
    "celebration": "medium",
    "adventure": "low",
    "cultural": "high",
    "workation": "low",
    "staycation": "low"
  },

  "travel_style_fit": {
    "cultural": "high",
    "scenic": "high",
    "luxury": "high",
    "relaxed": "medium",
    "adventure": "low",
    "nature": "low",
    "social": "medium",
    "road_trip": "medium"
  },

  "group_fit": {
    "solo": "high",
    "couple": "high",
    "friends": "high",
    "family": "high"
  },

  "experience_variety": "high",

  "known_for": ["Maharaja palaces", "Mehrangarh Fort", "Lake Pichola", "blue and pink city aesthetics", "local bazaars"],
  "not_suitable_for": ["beach", "nature or wildlife focus", "nightlife-centric trips"],

  "known_pinch_points": [
    "Summer heat (May–August) makes outdoor sightseeing very difficult",
    "December–January peak season: higher hotel prices, advance booking required",
    "Long internal drives if not using trains — cab-only adds fatigue"
  ],

  "experience_attributes": {
    "walkability": "medium",
    "food_culture": {
      "quality": "high",
      "accessibility": "high",
      "street_food": true,
      "notes": "Dal baati churma, laal maas, street food in bazaars. Easy to find without research."
    },
    "local_authenticity": "high",
    "tourist_density": "high",
    "spontaneity_friendly": "medium",
    "visual_character": "Forts, palaces, coloured cities, desert edge landscapes",
    "pace": "moderate",
    "discovery_potential": "high"
  },

  "meta": {
    "last_updated": "2025-10-01",
    "data_source": "curated",
    "confidence": "high"
  }
}
```

---

## Circuit Template Fields

| Field | Purpose in Meridian |
|---|---|
| `stops` | Step 1 candidate generation, Step 6 sequence validation, output |
| `internal_travel` | Step 5a travel-to-experience ratio, Step 6 validation, output |
| `duration` | Step 1 duration scoping |
| `gateway_connections` | Step 5 travel time feasibility, output entry section |
| `return_connection` | Step 6 return validation, output return section |
| `common_variations` | Step 5a substitution when ratio exceeded |
| `seasonality` | Step 2 rain/crowd filter, Step 7 seasonality scoring |
| `circuit_character` | Step 1 semantic search |
| `trip_goal_fit` | Step 3 goal fit elimination, Step 7 scoring |
| `travel_style_fit` | Step 7 travel style scoring |
| `group_fit` | Step 7 group fit scoring |
| `experience_variety` | Step 7 experience variety scoring (circuits only) |
| `known_pinch_points` | Output cons, refinement hooks |
| `experience_attributes` | Step 7 nuanced preference modifier |

Profiles are stored in two collections in a vector database (Pinecone / Weaviate / pgvector).

**Collection 1: Destination Profiles**
Individual destination profiles, vector embedded and indexed.

**Collection 2: Circuit Templates**
Pre-defined multi-stop templates, vector embedded and indexed separately. Queried by Meridian when `duration_nights` is 4 or more.

### Retrieval during Step 1

**Single destination query (1–3 nights)**
Semantic search on destination profiles using:
- Trip goal
- Travel style
- Origin city
- Trip duration

**Multi-stop query (4+ nights)**
Semantic search on circuit templates using:
- Trip goal
- Travel style
- Duration range match (`minimum_nights` ≤ `duration_nights` ≤ `maximum_nights`)
- Origin city (to assess entry/exit point feasibility)

Also query destination profiles for simpler 2-stop options where a full circuit template may not exist.

Structured filters applied on top of semantic results (both collections):
- Budget tier
- Travel month (`best_months`, `avoid_months`)
- Explicit exclusions

Nuanced preference matching applied as a scoring modifier after retrieval:
- Mapped KB experience attributes (`walkability`, `food_culture.accessibility`, `local_authenticity`, etc.)
- For circuits: assessed across all stops combined, not per stop individually

This combination — semantic retrieval for intent, structured filters for hard constraints, nuanced modifiers for scoring — gives Meridian a candidate pool that is relevant, constraint-compliant, and ranked by what the traveler actually cares about.

---

## Population Strategy

### Phase 0: Finalise schemas before generating any data

Lock both schemas — destination profile and circuit template — before generating a single entry. Populating before the schema is stable means regenerating everything when fields change.

---

### Phase 1: Seed circuit templates first

Seed circuits before individual destinations. Every stop in every circuit automatically becomes a Tier 1 destination — circuits tell you which destinations matter most. You are not guessing which 200 to cover.

**Step 1: Compile the master circuit list**

Target 40–60 circuits to start. This covers the vast majority of multi-stop queries at launch.

```
Rajasthan         →  Jaipur – Jodhpur – Udaipur (and extensions with Jaisalmer)
Golden Triangle   →  Delhi – Agra – Jaipur
Himachal          →  Shimla – Manali / Manali – Spiti / Kasol – Kheerganga
Kerala            →  Kochi – Munnar – Alleppey – Kovalam
Karnataka         →  Coorg – Mysore / Hampi – Gokarna / Bengaluru – Chikmagalur – Coorg
Northeast         →  Guwahati – Kaziranga – Shillong – Meghalaya
Uttarakhand       →  Rishikesh – Mussoorie / Chopta – Auli – Kedarnath
Odisha Coast      →  Bhubaneswar – Puri – Konark – Chilika
Maharashtra       →  Mumbai – Pune – Aurangabad (Ajanta/Ellora)
Madhya Pradesh    →  Bhopal – Khajuraho – Bandhavgarh
Andhra/Telangana  →  Hyderabad – Araku – Visakhapatnam
Tamil Nadu        →  Chennai – Pondicherry – Thanjavur – Madurai
```

**Step 2: For each circuit, define:**
- Canonical stop sequence and recommended night splits
- Total duration range (minimum, optimal, maximum)
- Best and avoid months for the circuit as a whole
- Internal travel legs with realistic durations and transport modes
- Circuit character and experience variety
- Known pinch points
- Common variations by duration

**Step 3: Human review (mandatory for circuits)**

LLMs consistently underestimate internal travel times and mischaracterise shoulder months for specific routes. Every circuit template must be reviewed by someone with direct knowledge of the route before it enters the KB.

---

### Phase 2: Seed individual destination profiles

Derive the priority list from circuit stops, then supplement.

**Tier 1 — Circuit stops**
Every stop from every Phase 1 circuit. Full profiles, human reviewed. These will appear in Meridian output most frequently.

**Tier 2 — High-traffic single destinations by origin city**
```
From Bengaluru   →  Coorg, Wayanad, Ooty, Hampi, Gokarna, Pondicherry, Chikmagalur
From Mumbai      →  Lonavala, Mahabaleshwar, Alibaug, Matheran, Kashid
From Delhi       →  Rishikesh, Mussoorie, Agra, Shimla, Nainital, Corbett
From Hyderabad   →  Warangal, Araku, Coorg, Hampi
From Chennai     →  Pondicherry, Ooty, Kodaikanal, Mahabalipuram, Varkala
From Pune        →  Lonavala, Mahabaleshwar, Kashid, Alibaug
```

**Tier 3 — Offbeat destinations**
Lower query frequency at launch but important for differentiation. Ziro, Tawang, Majuli, Chopta, Dholavira, Munsiyari, Gokarna (standalone). Add after Tier 1 and 2 are solid.

---

### Phase 3: Generation pipeline for each profile

**Step 1: LLM generation**
Feed the schema as a template. Instruct the model to populate every field and leave fields blank rather than guess. Use a strong model.

**Step 2: Web verification pass**
For each profile, run targeted searches to verify:
- Seasonality and monsoon character (travel blogs, local sources)
- Internal travel times for circuits (Google Maps, train schedules)
- Cost tier (current booking platforms)
- Crowd profile (recent visitor accounts, forums)

Typical LLM failure points: internal travel times (usually underestimated), shoulder month nuance, food accessibility for smaller towns, experience attributes.

**Step 3: Human review**
Someone with direct experience of the destination reviews the profile. Focus on: seasonality accuracy, travel time accuracy, experience attributes (these are hardest for LLMs to get right).

**Step 4: Confidence scoring**
```
LLM generated, not reviewed      →  confidence: low
LLM generated, web verified      →  confidence: medium
Human reviewed                   →  confidence: high
```

Meridian surfaces confidence in refinement hooks. A low-confidence profile can be used but flagged.

**Step 5: Store and index**
Vector embed the full profile. Store structured fields separately for hard filtering. Circuit templates stored in their own collection.

---

### Phase 4: Dynamic growth (post-launch)

When a destination or circuit not in the KB is queried:

```
Query not found in KB
    ↓
Generate profile — LLM + web research (automated)
    ↓
Store with confidence: low
    ↓
Use in current query
    ↓
Flag for human review after 3 queries
    ↓
Promote confidence after review
```

---

### Practical pre-launch timeline

| Task | Effort |
|---|---|
| Finalise both schemas | 1–2 days |
| Generate 50 circuit templates with LLM | 1 day |
| Human review of circuits | 2–3 days |
| Extract Tier 1 destination list from circuits | Half a day |
| Generate Tier 1 + Tier 2 destination profiles | 2 days |
| Web verification pass | 2–3 days |
| Human review of high-traffic profiles | 3–5 days |
| Store and index | 1 day |

**Total: approximately 2–3 weeks for a solid pre-launch KB.**

Dynamic growth handles everything beyond that.

---

## What the KB Does Not Cover

The KB stores what a destination *is*. The following are always fetched live:

| Data | Source |
|---|---|
| Current transport options and schedules | Maps API / transport APIs |
| Live travel time estimates | Maps API |
| Current pricing for stays and activities | Web search at query time |
| Road or weather conditions | Live data sources |

These change too frequently to cache reliably. Storing them in the KB would introduce stale data into budget and reachability outputs.
