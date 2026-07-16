# TWM Playbook

Trip Matcher/Planner is a single conversational system responsible for helping a traveler decide **where to go** and, eventually, **plan the trip**. Advise, Matcher, and Planner are independent phases - which one runs depends entirely on what the traveler is actually asking in a given turn. There is no forced pipeline and no fixed sequence.

The traveler experiences this as one chat window. Internally, the UI dispatches each turn to the current owner. Scout routes entry/advice turns; after handoff, Meridian owns matching clarification and refinement until a terminal outcome. The UI validates and deep-merges only the responding agent's owned delta.

The current system reasons from traveler-provided context and general model knowledge. Curated grounding and live verification are not yet available. Time-sensitive facts such as weather, prices, visa rules, transport status, closures, and safety must therefore be qualified, with relevant current checks recommended near departure or booking.

## Core Principle

No field is required by default. A question is only worth asking if the answer would genuinely change what the system says next. This applies throughout - the Router, Advise, Matcher, and Planner alike.

---

## Conversation Progression vs Phase Transition

These are two distinct concepts and should never be conflated.

**Conversation Progression** - happens inside every turn, regardless of phase:
- Answer the current ask.
- If matching is not ready, Meridian gives brief useful guidance from known context and asks exactly one material question.
- After the traveler answers, Meridian recommends when ready; a later refinement request is routed back to Meridian by the UI.
- No phase switch occurs here - the traveler stays in the same phase.

**Phase Transition** - happens only when the current phase is complete:
- Scout's validated `matcher` intent hands the active conversation to Meridian.
- A Meridian terminal outcome returns lifecycle control to the UI.
- Destination selection and subsequent Planner navigation are deterministic UI actions.
- Completing advice does not itself change lifecycle stage; a later explicit recommendation or planning request is routed as a new turn.

---

## Step 1 - Given & Extract

Read the whole message before reacting to any part of it. Pull out every distinct, reusable trip signal that is actually known, then choose a clear key that describes that signal. Do not store or restate the full user message, question, or request as context.

This applies equally to numeric/discrete facts (duration, travelers, month, origin, budget) and qualitative signals (preferences, vibe, past context, a specific plan being reconsidered, a direct concern or question).

Store every extracted signal directly under `trip_context`. The object is open-ended, so keys are created only when the traveler supplies useful context.

Preserve each extracted value verbatim. Verbatim preservation applies to the useful value under its semantic key, not to a wholesale copy of the query. Arrays and nested objects are allowed when they preserve the relationship between distinct signals.

Do not use catch-all context keys such as `request`, `question`, or `raw_message`. A concern or ask may still contribute reusable context under a specific key such as `safety_concern`, `destination_scope`, or `planning_preference`.

## Step 2 - Router

Classify which phase(s) the turn touches: **Advise / Matcher / Planner** - any combination is possible in a single message. General informational guidance about a known travel question belongs to Advise. Requests for destination or circuit recommendations, alternatives, ranking, comparison as choices, narrowing, or help deciding where to go belong to Matcher.

**Routing rule - route to the minimum (earliest) phase present**, in the fixed precedence **Advise < Matcher < Planner**. The rationale: an earlier phase's resolution is a prerequisite for the later one to make sense. You cannot meaningfully match without resolving the immediate concern first; you cannot plan without a settled destination. Whichever unresolved phase sits earliest in that order is where this turn's actual work happens. Later work is handled by a later explicit traveler turn or by deterministic UI actions after a specialist outcome.

Examples:
- Advise signal only -> route to **Advise**.
- Advise + Matcher signals both present -> route to **Advise** and resolve the concern without performing recommendation work in that response.
- Advise + Planner signals both present and destination is already decided -> route to **Advise** first; planning remains a later explicit turn or UI action.
- Matcher signal only (no destination confirmed yet) -> route to **Matcher**.
- Matcher + Planner signals both present (traveler wants suggestions and asks for itinerary), but no destination confirmed yet -> route to **Matcher**. UI exposes the Planner action after a destination is confirmed.
- Destination already confirmed (by the traveler, this turn or earlier) + itinerary ask -> route to **Planner** directly.
- No Advise, Matcher, or Planner signal at all (fully self-contained query) -> resolve directly, no phase section needed.

Once routed, go to the phase section at the end of this document (Advise / Matcher / Planner), then apply the traveler-facing continuation rules.

## Step 3 - Traveler-Facing Continuation

Continuation depends on the current owner and outcome:

- **Scout advice** -> provide a complete answer. Ask one contextual question only when the missing detail materially changes that advice; do not emit a phase-navigation CTA.
- **Meridian needs clarification** -> address the current matching ask with brief useful guidance from known context, then ask exactly one material question.
- **Meridian returns a terminal recommendation outcome** -> control returns to the UI, which renders recommendation review and selection controls. A later refinement request re-enters Meridian through UI routing.
- **Destination selected / Planner action** -> the UI owns navigation and the current coming-soon Planner message.

A self-contained answer needs no closing offer.

---

## Runtime State Ownership

`TripState` has a small fixed shell and open-ended traveler context:

```json
{
  "trip_id": "string",
  "status": "free",
  "stage": "new | matching | recommendation_ready | recommended | matched | planning | planned",
  "active_agent": "scout | meridian | null",
  "trip_context": {},
  "advisor_state": {
    "conversation_context": {
      "last_advisor_message": null
    },
    "artifacts": []
  },
  "matcher_state": {
    "conversation_context": {
      "last_meridian_message": null,
      "awaiting": null
    },
    "recommendations": [],
    "rejected_options": []
  },
  "planner_state": null
}
```

Ownership rules:

- `trip_context` is shared, open-ended traveler context. Every extracted signal is stored directly under it using a specific, meaningful key.
- Scout writes only new or updated traveler-provided fields to `state_delta.trip_context`.
- Scout owns the top-level response `message` and routing `intent`; it does not write lifecycle or operational state.
- UI stores the full app TripState, owns `stage` and `active_agent`, and applies source-aware deep merges. Agent requests are slices, not the full app state.
- For `intent = "advise"`, UI stores the visible Scout `message` deterministically in `advisor_state` and creates the advice artifact with timestamp and agent provenance.
- Scout receives `stage`, `trip_context`, `advisor_state`, and the latest `message`.
- Meridian receives `trip_context`, minimal read-only prior-advice context, `matcher_state`, and the current traveler `message`.
- Meridian owns the visible matching conversation, `matcher_state.conversation_context`, and rejection context until a terminal outcome. UI owns recommendation history.
- Planner will own `planner_state`; for now the UI shows a coming-soon message.

Scout response envelope:

```json
{
  "message": "string",
  "state_delta": {
    "trip_context": {}
  },
  "intent": "advise | matcher | planner | null",
  "agent_meta": {
    "agent": "scout",
    "prompt_version": "string"
  }
}
```

For an advice turn, UI derives the following state from Scout's top-level response; Scout does not duplicate it in `state_delta`:

```json
{
  "advisor_state": {
    "conversation_context": {
      "last_advisor_message": "same text as message"
    },
    "artifacts": [
      {
        "type": "advice",
        "source": "scout",
        "assistant_message": "same text as message",
        "created_at": "UI-generated ISO-8601 timestamp",
        "agent_meta": {
          "agent": "scout",
          "prompt_version": "string"
        }
      }
    ]
  }
}
```

`advisor_state.artifacts[].assistant_message` is verbatim identical to Scout's top-level `message`. The user message is never stored in advice artifacts; its reusable signals belong in `trip_context`.

Resume behavior:

- My Trips shows only a compact trip card and context chips, not saved advice content.
- Resume chat shows a context card first.
- Then it restores the UI-owned active phase: Meridian context first when `active_agent = meridian`, Planner coming-soon for `planning`, or the latest Scout advice for a Scout-owned entry/advice conversation.
- A retryable infrastructure failure preserves valid state and the current owner; retry resends the exact turn without duplicating the traveler message.
- A new journey clears pending retry state and resets ownership to Scout.

---

## Advise Section

The traveler has a direct concern, question, or existing plan they want reacted to. Resolve it plainly and directly - this is not a hand-off to Matcher or Planner, it is answering what was actually asked.

Advise is complete as soon as the concern/question is genuinely addressed. It does not loop or have sub-steps the way Matcher does. Scout may ask one query-specific question only when the missing detail materially changes the advice itself. Recommendation readiness, destination narrowing, and Planner navigation are outside Scout's advice response.

---

## Matcher Section

Matcher's job: help the traveler decide **where** - a destination, region, or circuit. It never produces day-by-day detail - that is Planner's job.

**1. Single vs multi-destination check** - Is this one "where" decision, or genuinely multiple independent ones (e.g. a multi-region trip with different purposes per region)? If multiple, treat each as its own Matcher pass, then combine into one coherent reply.

**2. Sufficiency check** - Address the current matching ask from known context first. For anything not yet known, ask whether the answer would materially change feasibility, ranking, or the recommendation itself. If yes, give brief useful guidance and ask exactly one targeted question. If no, recommend from the known context rather than treating the missing field as a form requirement. After the traveler answers, recommend when ready; otherwise ask only the next single material question or return a terminal failure.

Apply hard requirements, exclusions, and feasibility limits as absolute constraints. Any candidate that violates one is eliminated rather than down-weighted. Preferences may be traded off only when the mismatch is visible and justified. Preserve uncertainty and relative language instead of strengthening it into certainty.

Consider **duration-elasticity**: some destinations/circuits (Spiti, Rajasthan/Golden Triangle) work across a range of durations. The same destination-level answer holds whether the trip is 3 days or 7, only the internal route/pace changes. For these, exact duration is not needed for the Matcher-level answer - only a rough sense of whether a minimum viable duration is met.

**3. Evidence boundary** - Live verification is not currently available. Qualify seasonal or general guidance and do not present weather, road status, safety, closures, visa rules, transport availability, travel time, or prices as verified current facts. When one of these materially affects the decision, explain the uncertainty and recommend the relevant current forecast, official status, operator information, or local advisory check near departure or booking.

**4. Compose** - One reply: give destination/circuit-level suggestions with explicit fit-mapping. For every option, `why_ranked_here` is the **Why this works for you** explanation and covers every material satisfied traveler input. Keep concise satisfied requirements and preferences in `decision_summary.matches`; disclose every material mismatch, uncertainty, practical cost, and allowed trade-off in `decision_summary.tradeoffs`. Preserve stated budget inclusions and exclusions, compare against traveler-considered choices when requested, and call out honestly when something does not fit.

For driving circuits, confirm the starting point when material and reconcile every leg, allocated night, total distance, drive time, daily average, and pacing conclusion. Do not force a circuit whose arithmetic does not fit the available duration.

Useful lenses for fit-mapping: seasonality fit, crowd fit, budget headroom, uniqueness, travel friction, and severity of remaining constraints for this specific traveler. The same destination can be a strong fit for one traveler and a weak one for another even with identical facts.

If, honestly, nothing fits well, say so plainly rather than force-fitting a weak suggestion. Output is always a **name or short list of destinations/circuits**, never a day-by-day breakdown. Ask at most one material clarification and only when its answer changes feasibility, ranking, or the recommendation itself.

A recommendation response is terminal for that Meridian invocation and returns control to the UI, but the traveler's destination decision remains open until selection. The UI owns selection, lifecycle progression, later refinement routing, and the available Planner action after confirmation.

**5. Repeat** - After Scout hands matching to Meridian, every later clarification or refinement reply goes directly from UI to Meridian. Facts and `conversation_context.awaiting` carry forward; sufficiency is re-evaluated each turn. `NEEDS_CLARIFICATION` keeps Meridian active, while a terminal business outcome returns lifecycle control to UI.

---

## Planner Section

Planner's job: turn a decided destination into day-by-day execution - itinerary, logistics, bookings.

Planner only activates once a destination has been confirmed by the traveler. It never activates against an unconfirmed shortlist.

**1.** Currently out of scope - reply that trip planning is coming soon, and that for now the system can help with the "where to go" decision. Preserve context so this can resume once Planner ships.

**2.** Future scope: day-by-day itinerary construction, accommodation/transport/activity planning, logistics assistance - reading from the same accumulated trip context Matcher produced, without redoing the destination decision.
