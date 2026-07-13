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

## Product Documentation

- [Architecture](ARCHITECTURE.md)
- [TripState](TRIP_STATE.md)
- [Lifecycle stage transitions](STAGE_TRANSITIONS.md)
- [Trip Matcher](trip-matcher/README.md)
- [Trip Matcher API contracts](trip-matcher/API_CONTRACTS.md)
