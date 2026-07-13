# Trip Matcher

Trip Matcher/Planner is the single conversational travel front door for TravelWithMe.

Its job is to help a traveler ask for advice, decide where to go, and eventually plan the trip from one chat surface.

The success metric is simple:

```text
The traveler gets a useful answer for the current ask and, when relevant, reaches a confident destination or circuit decision grounded in their own context.
```

## Scope

Current scope focuses on:

```text
- extracting raw trip context from the traveler
- routing each turn to Advise, Matcher, Planner, or no phase
- answering advice turns directly
- calling Meridian automatically for Matcher turns
- preserving context for planner handoff
```

## Agents

The system is split by responsibility:

```text
Scout
  -> conversational front door
  -> extracts trip_context
  -> routes turns with intent = advise | matcher | planner | null
  -> answers Advise turns
  -> leaves Matcher and Planner visible replies to downstream owners
```

[Scout](SCOUT.md)

```
Meridian
  -> conversational matcher
  -> evaluates open-ended trip_context
  -> asks one soft clarification when needed
  -> returns destination or circuit matches with message and state_delta
```
[Meridian](MERIDIAN.md)

## Flow

```text
Traveler talks to Scout
  -> Scout extracts trip_context
  -> Scout returns intent
  -> UI merges state_delta into TripState
  -> if intent = advise, Scout answers directly and UI stores the visible advice memory
  -> if intent = matcher, UI calls Meridian in the same chat
  -> if intent = planner, UI/planner layer returns the temporary planner coming-soon response
```

TripState is the shared source of truth. `trip_context` starts as `{}` and stays open-ended. Scout stores every useful extracted signal directly under it using a specific key. Values preserve the traveler's wording where possible, but the full user query is not copied into context.

## API Contracts

See [API contracts](API_CONTRACTS.md).

## Architecture And Operations

- [Architecture](../ARCHITECTURE.md)
- [Current EC2 deployment](../EC2_SETUP.md)
- [n8n setup and workflow operations](../SELF_HOSTED_N8N.md)
