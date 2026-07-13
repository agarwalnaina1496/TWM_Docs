# Travel With Me

Travel With Me (TWM) uses Scout as the conversational front door for every traveler query. Scout preserves useful trip context, answers advice turns, and routes work to the appropriate product capability.

## Conversational Flow

```mermaid
flowchart TB
    User["Traveler sends a query"]

    subgraph UI["TWM UI"]
        Build["Build Scout request<br/>stage + trip_context + advisor_state + message"]
        Merge["Merge Scout state_delta<br/>and preserve TripState"]
        Intent{"Check Scout intent"}
    end

    subgraph API["FastAPI on Render"]
        ScoutAPI["POST /scout"]
        ScoutEngine["Load active Scout prompt<br/>and call N8NAgentEngine"]
        Normalize["Normalize response<br/>and attach trusted agent_meta"]
    end

    subgraph Runtime["n8n on EC2"]
        ScoutWorkflow["scout.json webhook"]
        ScoutAgent["Scout<br/>extract context + route turn"]
        ScoutParser["Parse structured response"]
        ScoutReply["Return webhook response"]

        ScoutWorkflow --> ScoutAgent --> ScoutParser --> ScoutReply
    end

    Advice["Advise<br/>UI stores and shows Scout reply"]
    Matcher["Trip Matcher<br/>Agent: Meridian"]
    Planner["Trip Planner<br/>UI-owned coming-soon response"]
    Direct["No phase<br/>Show Scout reply when present"]

    User --> Build --> ScoutAPI
    ScoutAPI --> ScoutEngine --> ScoutWorkflow
    ScoutReply --> ScoutEngine
    ScoutEngine --> Normalize --> Merge --> Intent

    Intent -->|advise| Advice --> User
    Intent -->|matcher| Matcher --> User
    Intent -->|planner| Planner --> User
    Intent -->|null| Direct --> User
```

Scout does not generate destination rankings. When Scout returns `intent = matcher`, the UI calls the Trip Matcher in the same chat turn. Meridian is the agent responsible for the matcher response.

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

    Advise["intent = advise<br/>Scout answers directly<br/>and may add a contextual CTA"]
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
