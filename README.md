# Travel With Me

Travel With Me (TWM) uses Scout as the conversational front door for a new journey and for Scout-owned advice. After Scout hands a phase to a specialist, the UI routes later turns directly to that active specialist until the specialist returns a terminal outcome or the traveler starts a new journey.

## Conversational Flow

```mermaid
flowchart TB
    User["Traveler sends a turn"]
    Dispatch{"UI active-agent dispatcher"}
    Scout["Scout entry or advice turn<br/>POST /scout"]
    ScoutValid{"Valid Scout result?"}
    ScoutRetry["Keep state and owner<br/>offer retry for the same turn"]
    ScoutMerge["UI merges Scout-owned<br/>trip_context delta"]
    Intent{"Scout intent"}
    Advice["advise or null<br/>UI stores/shows Scout reply"]
    Handoff["matcher<br/>UI sets active_agent = meridian<br/>and builds the deep-merged phase slice"]
    Meridian["Meridian initial or continued turn<br/>POST /meridian with message"]
    MeridianValid{"Valid Meridian result?"}
    MeridianRetry["Keep matching owner and state<br/>offer retry for the same turn"]
    MeridianMerge["UI merges Meridian-owned delta"]
    Outcome{"Meridian outcome"}
    Continue["NEEDS_CLARIFICATION<br/>keep active_agent = meridian"]
    Terminal["Terminal business outcome<br/>clear active specialist<br/>UI stores/presents result"]
    Planner["planner<br/>UI shows deterministic<br/>coming-soon placeholder"]
    Reset["New journey<br/>UI resets owner to Scout"]

    User --> Dispatch
    Dispatch -->|Scout owns turn| Scout --> ScoutValid
    Dispatch -->|Meridian owns matching| Meridian
    ScoutValid -->|No| ScoutRetry --> User
    ScoutValid -->|Yes| ScoutMerge --> Intent
    Intent -->|advise or null| Advice --> User
    Intent -->|matcher| Handoff --> Meridian
    Intent -->|planner| Planner --> User
    Meridian --> MeridianValid
    MeridianValid -->|No| MeridianRetry --> User
    MeridianValid -->|Yes| MeridianMerge --> Outcome
    Outcome -->|continuing| Continue --> User
    Outcome -->|terminal| Terminal --> User
    User -->|new journey action| Reset
    Reset --> Dispatch
```

The UI owns `active_agent`, lifecycle `stage`, persistence, retry, selection, navigation, and recommendation history. Scout and Meridian return only their validated messages, routing or business outcome, and agent-owned deltas.

Scout does not generate destination rankings. When Scout returns `intent = matcher`, the UI first merges Scout's traveler-context delta, then calls Meridian with the updated phase slice and the same traveler message. Later matching replies go directly to Meridian without returning through Scout.

See the [Trip Matcher flow](trip-matcher/README.md) for the complete Meridian request, execution, and response lifecycle.

## Scout Internal Decision Flow

Scout extracts traveler context before deciding who should respond. It routes to the earliest unresolved phase and returns only context added or changed by the current turn.

```mermaid
flowchart TB
    Input["Latest message + Scout state slice<br/>stage + trip_context + advisor_state"]
    HasMessage{"Is message present?"}
    Extract["Read the complete message<br/>extract every reusable traveler signal"]
    Delta["Build state_delta.trip_context<br/>new or updated traveler-provided values only"]
    Resume["Resume from existing context<br/>without extracting new values"]
    Detect["Detect phase signals<br/>advise + matcher + planner"]
    Confirm["Evaluate destination confirmation<br/>selected_option or explicit traveler choice"]
    Advice{"Advice concern, comparison,<br/>question, or doubt?"}
    Match{"Destination decision needed,<br/>or planning requested without<br/>a confirmed destination?"}
    Plan{"Planning requested with<br/>a confirmed destination?"}

    Advise["intent = advise<br/>Scout answers directly<br/>and may ask a tailored follow-up"]
    Matcher["intent = matcher<br/>Scout preserves context only<br/>Meridian owns the visible response"]
    Planner["intent = planner<br/>Scout does not create an itinerary<br/>Planner/UI owns the visible response"]
    Direct["intent = null<br/>Scout answers the self-contained query"]
    Output["Return structured Scout result<br/>message + state_delta.trip_context + intent<br/>never stage or destination rankings"]

    Input --> HasMessage
    HasMessage -->|yes| Extract --> Delta --> Detect
    HasMessage -->|no| Resume --> Detect
    Detect --> Confirm --> Advice
    Advice -->|yes| Advise --> Output
    Advice -->|no| Match
    Match -->|yes| Matcher --> Output
    Match -->|no| Plan
    Plan -->|yes| Planner --> Output
    Plan -->|no| Direct --> Output
```

The routing order is `advise → matcher → planner`: when a turn touches multiple phases, Scout selects the earliest phase that is still unresolved. A casually mentioned destination is not confirmation; a deterministic `trip_context.selected_option` or an explicit traveler choice is.

## Product Documentation

- [Architecture](ARCHITECTURE.md)
- [TripState](TRIP_STATE.md)
- [Lifecycle stage transitions](STAGE_TRANSITIONS.md)
- [Trip Matcher](trip-matcher/README.md)
- [Trip Matcher API contracts](trip-matcher/API_CONTRACTS.md)
